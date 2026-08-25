# Datus-Agent 知识库 RAG 与语义层深度问答

> 本文档由一次完整技术问答整理而来，内容基于仓库真实代码与文档（附 `文件:行号` 可查证），覆盖：
> 1. 四类知识的 Embedding / 精准检索 / RAG 应用
> 2. 自动化创建流程
> 3. 如何保持最新（含删减一致性）
> 4. 总体设计 / 详细设计 / 实现细节 / 难点与解法（由浅入深）
> 5. 手动与自动的边界
> 6. 项目复杂度 / 语义模型覆盖 / 检索准确性 / NL2SQL 准确性评估
> 7. MetricFlow 与 OSI 的定位、规格、编译器
> 8. 编写规格 / 执行后端 / 编译器在项目中的功能与流程

---

## 一、总体设计

### 1.1 一句话

> 这是一个**「多路检索的 RAG 知识库」**：把数据库结构、业务语义、历史成功 SQL、可复用模板四类知识，在**写入时自动向量化**存进 LanceDB，Agent 在生成 SQL 前用「向量 + 全文 + 标量过滤」三路召回，喂给 LLM 作为上下文。

### 1.2 数据在哪儿、长什么样

- 每个项目一个 LanceDB 目录：`~/.datus/data/{project}/datus_db/`，里面是一张张**向量表**：`schema_metadata`、`schema_value`、`semantic_model`、`reference_sql`、`reference_template`、`table_semantic_profile`、`metric` 等。
- 每张表有固定的**列结构（schema）**，其中指定一列是**向量来源列**（`vector_source_name`），写入时这一列的文本会被 embedding 成向量存进 `vector` 列。
- 每行还带**作用域列**（`datasource_id`），实现多数据源/多租户行级隔离。

### 1.3 分层架构图

```
┌────────────────────────────────────────────────────────┐
│ 消费层：Agent 工具（schema linking / 指标生成 / SQL生成）  │
│   database.py:search_similar → 表DDL+样例行 → prompt     │
│   sql_summary_agentic_node: 检索相似 reference SQL       │
└──────────────▲─────────────────────────────────────────┘
┌──────────────┴─────────────────────────────────────────┐
│ RAG 接口层：SchemaWithValueRAG / SemanticModelRAG /      │
│   ReferenceSqlRAG / ReferenceTemplateRAG / MetadataFtsRAG│
│   统一：store_batch/truncate/search_*/get_schema/        │
│   after_init；自动加 datasource_id + sub-agent 过滤      │
└──────────────▲─────────────────────────────────────────┘
┌──────────────┴─────────────────────────────────────────┐
│ 存储层：BaseEmbeddingStore（写入即embed、检索、建索引）    │
│   VectorTable/Database 抽象 → LanceDB 实现               │
└──────────────▲─────────────────────────────────────────┘
┌──────────────┴─────────────────────────────────────────┐
│ Embedding 层：EmbeddingModel（FastEmbed本地 / OpenAI云） │
└────────────────────────────────────────────────────────┘
构建入口：bootstrap-kb（metadata/semantic_model/metrics/
          reference_sql/reference_template）
```

### 1.4 两条时间线

- **离线构建（bootstrap-kb）**：手动/脚本触发，批量灌数据 + 建索引。
- **运行时（Agent 会话）**：写侧自动同步生成物、后台增量刷新；读侧自动检索。

---

## 二、详细设计

### 2.1 Embedding 模型设计（embedding_models.py）

- `EmbeddingModel` 一个类封装两种后端：
  - **本地 FastEmbed**（`registry_name=sentence-transformers/fastembed`），默认 `all-MiniLM-L6-v2` 384 维；
  - **云端 OpenAI**（`embedding_openai.py`），通过 `agent.models` 里的配置拿 key/base_url。
- 按 store 名注册（`database`/`document`/`metric`/`semantic_model`/…），`get_embedding_model(name)` 取用；在 `agent.yml: storage:` 段可配每类模型、维度、batch_size、设备（`embedding_device_type`，默认自动选 cpu/gpu/mps）。
- **懒加载 + 锁 + 失败标记**：第一次用时才下载/加载，失败后 `is_model_failed` 置位，后续直接抛 `ErrorCode.MODEL_EMBEDDING_ERROR`；`has_local_fastembed_snapshot()` 检查本地缓存避免搜索时联网下载。

### 2.2 存储层设计（base.py + lance_backend.py）

- `BaseEmbeddingStore.__init__` 接收：表名、embedding 模型、schema、`vector_source_name`（写哪列生成向量）、`vector_column_name`、唯一键、是否 `datasource_scoped`。
- **写入即 embed**：建表时给 LanceDB 传 `EmbeddingFunctionConfig(vector_column, source_column, function)`（lance_backend.py:376），后续 `add/merge_insert` 缺 vector 列时由 LanceDB 自动调 `generate_embeddings(source_column)`。代码里没有任何手动算向量的地方。
- 三类索引（`create_indices`）：
  - **向量索引**：cosine，IVF_PQ（≥5000 行）/IVF_FLAT，partition 数按 `sqrt(row_count)` 自适应；
  - **FTS 全文索引**：每张表配 `FtsSpec(field, boost)`；
  - **标量索引**：`kind/table_name/database_name/…`，加速过滤。
- 检索三模式（`search()`，base.py:581）：`vector`（语义）、`hybrid`（向量+FTS，用 `LinearCombinationReranker` 重排，base.py:610）、`fts`。hybrid 失败自动降级纯向量。
- 并发写：进程内 `_write_lock` + LanceDB 提交冲突时**重试/指数退避**（`_add_with_retry/_upsert_with_retry`，base.py:546）。

### 2.3 每类知识的「向量来源列」（关键差异表）

| 知识 | 表 | 向量来源列（embed 什么） | 检索时怎么用 |
|---|---|---|---|
| 表结构 | `schema_metadata` | `definition`（CREATE TABLE DDL） | search_similar → prompt |
| 样例值 | `schema_value` | `sample_rows`（top5 样例行 CSV） | 与 schema 一起喂 prompt，帮 LLM 猜值/过滤 |
| 语义模型 | `semantic_model` | `description`（表/列描述，逐行一个对象，id 如 `table:orders`/`column:orders.amount`） | 检索+精确取表（get_semantic_model 重组 dimensions/measures） |
| Reference SQL | `reference_sql` | `search_text`（LLM 生成的 name+summary+tags+SQL 拼接文本） | 相似问题→召回历史成功 SQL |
| Reference Template | `reference_template` | `search_text`（模板摘要+参数） | 相似场景→召回可参数化模板 |
| 表语义 profile | `table_semantic_profile` | `search_text`（MetricFlow/OSI 文档投影） | 供 describe_table 等 DB 工具 |
| 指标 | `metric` | （类似） | 指标语义检索 |

### 2.4 RAG 接口层（store.py 里每个 `*RAG` 类）

- 统一接口：`store_batch/upsert_batch`（写）、`truncate`（清本数据源）、`search_*`/`get_schema`/`search_tables`（读）、`after_init`（建索引）、`get_*_size`（统计）。
- **自动注入作用域**：所有读自动拼 `datasource_condition(datasource_id)`；多租户场景还有 `_build_sub_agent_filter`（rag_scope.py）按 sub-agent 的 `tables`/`sqls`/`metrics` 范围过滤，防止越权取数。

### 2.5 精准搜索的「三路 + 精确回退」

1. **语义**（vector）：自然语言 → 向量余弦相似度。
2. **关键词**（FTS）：`SemanticModelStorage.create_indices` 配 boost（name/fq_name 3×、description 1×）；纯 FTS 模式走 `MetadataFtsRAG`（kb_retrieval/store.py:432，ngram 文档表，把 schema+profile+样例行拼成一个 `search_text` 再 FTS）。
3. **标量过滤**：catalog/database/schema/table/`kind`/`table_type`/datasource_id/subject 树节点。
4. **精确回退**：`get_schema`/`search_tables` 用等值过滤精确取某张表；`_identifier_variants`/`_normalized_identifier`/`_hierarchy_compatible`（semantic_model/store.py:33）处理引号、大小写、库名/库别名/多级限定名歧义。
5. **subject 树过滤**：reference SQL/template 按 `SubjectTreeStore`（RDB 邻接表）分层，`get_matched_children_id` 支持 `*` 通配与子代递归，转成 `subject_node_id IN (...)` 再向量搜（subject_tree/store.py:814）。

### 2.6 RAG 应用（谁在消费）

- **Schema Linking**：`database.py:1069` `search_similar` → `TableSchema/TableValue` → 拼进 LLM 上下文；`schema_lineage.py` 做表关联度排序。
- **参考 SQL/模板复用**：`sql_summary_agentic_node.py:206` 生成摘要时先检索相似 reference SQL 作为 few-shot。
- **指标/语义模型**：生成指标前按表取语义模型、按 subject 取已有指标。

---

## 三、实现细节：自动化创建

### 3.1 编排入口（agent.py:393 `bootstrap_kb`）

`datus-agent bootstrap-kb --components metadata,semantic_model,metrics,reference_sql,reference_template --kb_update_strategy overwrite|incremental|check|refresh-profile`

策略语义：
- `overwrite`：先删 YAML 源目录 + `truncate()` 清向量库 → 全量重建。
- `incremental`：保留旧数据，按 id/identifier diff 只处理增量。
- `check`：只统计不生成。
- `refresh-profile`：仅刷新表语义 profile。

### 3.2 metadata（local_init.py）

1. 通过 db_manager 连接库 → `get_tables_with_ddl()` 拿 DDL（表/视图/MV 分类型）。
2. `exists_table_value`（init_utils.py:15）读出库里已存的 `identifier→definition` 与有值的表集合。
3. `store_tables`（local_init.py:414）**逐表 diff**：新增→写；`definition` 变了→先 `remove_data` 删旧行再重写；没变→跳过。
4. `_fill_sample_rows`（:519）对新增/变更表取 top5 样例行 → `schema_value` 表。
5. `after_init` 建索引。

### 3.3 semantic model（semantic_model_init.py）

三种来源，最终都落到「写 YAML → `_sync_semantic_to_db` 逐对象 upsert 向量库」：
- **Success story CSV**：整个 CSV 一次性喂 `GenSemanticModelAgenticNode`（工作流模式），为 SQL 出现的表生成语义模型 YAML（`subject/semantic_models/*.yaml`）。
- **adapter**：`init_from_adapter` 从 MetricFlow/OSI 适配器拉取。
- **已有 YAML**：`init_semantic_yaml_semantic_model` 直接解析导入。

### 3.4 reference SQL（reference_sql_init.py + sql_file_processor.py）

1. 扫描 `--sql_dir` 的 `.sql`，`parse_comment_sql_pairs` 按**分号切块**（状态机跳过注释/字符串，保证块正确）。
2. `sqlglot` 三方言（mysql/hive/spark）校验，**只保留 SELECT**，并 pretty 规范化。
3. 逐条喂 `SqlSummaryAgenticNode`，LLM 生成 `name/summary/search_text/subject_tree/tags`，写 `subject/sql_summaries/*.yaml`。
4. 立即 `_sync_reference_sql_to_db` 按 `id=md5(sql)` upsert 入库。

### 3.5 reference template（同 reference_sql 流程 + 参数分析）

- jinja2 AST 校验 + `extract_template_parameters` 提 `{{var}}`。
- `analyze_template_parameters`（template_file_processor.py:43）用**sqlglot 静态分析**推断参数类型：`dimension`（WHERE col='{{x}}'，解析成 `table.column`）、`column`（GROUP BY/SELECT）、`keyword`（ORDER BY → ASC/DESC）、`number`（LIMIT/比较）。
- 运行时还可用 `_enrich_dimension_sample_values` 从库里查参数常见值（只对安全标识符做）。

### 3.6 metrics / table profile

- 依赖已建语义模型，`init_success_story_metrics` 生成指标；`_sync_table_semantic_profiles` 从 MetricFlow/OSI 文档投影出物理表维度 profile。

---

## 四、实现细节：如何保持最新

| 机制 | 触发 | 实现 |
|---|---|---|
| 可重入 bootstrap | 手动 | overwrite 全量 / incremental 按 id/identifier diff |
| 会话内即时同步 | `SqlSummaryAgenticNode._save_to_db`（sql_summary_agentic_node.py:507） | 每次生成 YAML 后自动 `_sync_reference_sql_to_db/template`，md5 id 去重 upsert |
| 生成物自动入库 | `GenerationHooks` 工具结束钩子 | 同步语义对象/profile/指标 + artifact replacement 清理 |
| 后台增量刷新 | `BackgroundSchemaSyncManager`（background_sync.py） | 启动/切数据源时 `init_local_schema_async(incremental)`；单槽调度、可取消、drift guard、失败仅告警；完成后刷新 @Table 补全缓存 |
| 索引增量维护 | `after_init(incremental)` / `optimize()` | LanceDB 增量合并新 fragment，不重建全量；FTS 用 `check_ready`+`FtsIndexStatus` 校验版本 |
| 同工件一致性 | artifact replacement（artifact_replacement.py） | 见下 |

**Artifact replacement（删减一致性的核心）**，如 `_sync_osi_semantic_to_db`（generation_tools.py:1969-1991）：

```
snapshot_artifact_replacements(快照该 yaml_path 旧行)
→ semantic_rag.upsert_batch(新对象)
→ delete_stale_artifact_rows: delete_artifact_rows_except(yaml_path, keep_ids)
     删除该工件下不在新 id 集合的旧行
→ delete_shadowed_table_rows: 清理跨工件、被新 id 遮蔽的遗留行
失败 → restore_artifact_replacements(按快照回滚)
```

表语义 profile 删除行时会 `_refresh_metadata_documents_for_tables` 联动重建 metadata FTS 文档（kb_retrieval/store.py:817），这是「元数据 ↔ 语义模型」的正向联动点。

---

## 五、难点在哪里？怎么解决的

### 难点 1：语义 vs 精确的平衡
**问题**：自然语言查询命不中表名，精确表名又该精准命中，二者冲突。
**解法**：三路检索并存——向量（语义）、FTS+boost（关键词精确）、标量过滤（结构精确）；hybrid 用 `LinearCombinationReranker` 融合；`get_schema`/`search_tables` 保底精确取表。

### 难点 2：SQL 标识符歧义
**问题**：`Orders` vs `orders` vs `"orders"` vs `db.schema.Orders`、不同 schema 同名表。
**解法**：`_identifier_variants` 生成引号剥离/大小写/多级限定名变体（semantic_model/store.py:33）；`_hierarchy_compatible` 空值兼容、`_normalized_identifier` 归一；`table_exists` 先等值快查再窄字段扫描兜底。

### 难点 3：多租户/多数据源隔离
**问题**：一个项目多数据源，检索串数据。
**解法**：`datasource_id` 行级列 + 所有 RAG 读路径自动拼 `datasource_condition`；sub-agent 再叠 `ScopedFilterBuilder` 的表/主题过滤；物理表仍是项目级共享（`datasource_scoped` 表设计，base.py:105）。

### 难点 4：增量更新不重复
**问题**：重复跑 bootstrap 会重复入库；内容变了要更新。
**解法**：`id=md5(sql)`/`storage_key` 唯一键 + `merge_insert` upsert；metadata 用 `identifier→definition` 全量比对，变了先删后写；`exists_*`（init_utils 系列）在增量模式下跳过已存在条目。

### 难点 5：生成物与向量库的一致性（含删减）
**问题**：LLM 改了一个语义模型 YAML，库里旧行不删 → 影子数据/脏数据。
**解法**：**Artifact replacement**（snapshot → upsert → `delete_artifact_rows_except` → `delete_shadowed_table_rows`），失败自动回滚；profile/metrics 同模式；YAML 是权威源，向量库是它的投影。

### 难点 6：删减的缺口（当前设计局限）
**问题**：表从库里删除/改名，`store_tables` 只增改**不删**「库里已消失的表」；没有自动 GC 级联清理语义模型。
**解法（现方案）**：`overwrite` 全量重建兜底（先删 YAML + truncate）；会话内 `ensure_semantic_models_exist`（auto_create.py:377）对 SQL 引用的缺失表按需自动补建（前向一致性）。**这是已知待改进点**：删除需要手动触发。

### 难点 7：embedding 模型可用性
**问题**：模型下载慢、失败、搜索时联网下载会卡。
**解法**：懒加载 + 锁 + `is_model_failed` 失败标记；`has_local_fastembed_snapshot()` 检查本地快照，`_ensure_embedding_cache_ready_for_search` 在搜索前拦截；后台同步对非关键任务跳过「缺缓存」场景（background_sync.py:194）。

### 难点 8：并发写冲突
**问题**：多线程/多进程写 LanceDB 触发 `Commit conflict`。
**解法**：进程内写锁 + `_add_with_retry/_upsert_with_retry`（3 次、指数退避、刷新表句柄重试，base.py:546）。

### 难点 9：SQL/模板文件解析健壮性
**问题**：分号在注释/字符串里、多方言、Jinja2 块嵌套，切错块会入库脏数据。
**解法**：手写状态机 `_find_effective_semicolon`（sql_file_processor.py:97）跳过 `--`/`/* */`/引号；模板用 Jinja2 块深度计数（template_file_processor.py:318）；sqlglot 多方言校验，非法条目写日志不阻断。

### 难点 10：LLM 生成摘要质量不稳定
**问题**：reference SQL 的 `search_text` 决定召回质量。
**解法**：强制 LLM 输出 `name/summary/search_text/subject_tree/tags` 结构化字段；模板参数由**静态 sqlglot 分析**确定性推断（不靠 LLM 猜）；把 `search_text` 设计成业务意图 + 参数名的拼接文本，再对 name/summary 加权 boost。

### 难点 11：索引与数据版本漂移
**问题**：旧 FTS 索引（Tantivy）与新配置不匹配，搜不到。
**解法**：`FtsIndexStatus`（MISSING/LEGACY/VERSION_MISMATCH/READY）+ `remove_legacy_fts_index` + `require_fts_ready`，bootstrap 失败时报错提示 `--kb_update_strategy overwrite` 重建（kb_retrieval/store.py:301）。

### 难点 12：后台同步与主线程竞态
**问题**：用户快速切换数据源，后台同步跑旧数据源或堆积。
**解法**：`BackgroundSchemaSyncManager` 单槽调度——新请求**取消**旧的、同数据源合并、`is_running` 线程安全、执行前 `drift guard`（当前数据源变了就放弃，background_sync.py:144）。

---

## 六、手动 vs 自动

### 手动（用户/CLI 触发）

| 操作 | 说明 |
|---|---|
| `bootstrap-kb --components ... --kb_update_strategy overwrite\|incremental\|check\|refresh-profile` | 全量/增量/检查/刷新 profile，构建 metadata、semantic_model、metrics、reference_sql、reference_template（agent.py:393） |
| 直接编辑 `subject/semantic_models`、`subject/sql_summaries`、模板 YAML 源文件 | **没有文件 watcher**，改完不会自动同步，需重新 bootstrap 或走工具 |
| CLI subject 面板增/删/改 metric、reference SQL（subject_screen.py:1010/2010） | 经 `SubjectUpdater.update_entry/delete` 操作向量库，并自动把改动同步回 YAML——半自动（你操作，代码保证两侧一致） |
| 运维：删缓存、手动 truncate、overwrite 兜底重建 | 手动 |

### 自动（运行时 / 后台）

| 环节 | 触发点 | 机制 |
|---|---|---|
| 写入即 embed | 任何 store 写操作 | LanceDB `EmbeddingFunctionConfig` 绑定源列，merge/add 时自动调 `generate_embeddings`（lance_backend.py:376） |
| 会话内即时同步 | `SqlSummaryAgenticNode._save_to_db`（sql_summary_agentic_node.py:507） | 每次成功生成 SQL summary/模板 YAML 后自动 `_sync_reference_sql_to_db`/`_sync_reference_template_to_db`，按 md5 id 去重 upsert |
| 生成物自动入库 | `GenerationHooks` 工具结束钩子（generation_hooks.py:293） | agent 调用生成类工具后自动同步语义对象/profile/指标，并做 artifact replacement 清理 |
| 后台元数据刷新 | `BackgroundSchemaSyncManager`（background_sync.py） | 启动/切换 datasource 时自动跑 `init_local_schema_async(incremental)`，单槽、可取消、失败仅告警 |
| 索引维护 | `after_init` / `optimize` | 自动建向量+FTS+标量索引，incremental 自动增量合并 fragment |
| 同工件一致性 | artifact replacement（artifact_replacement.py） | 同步时自动 snapshot→upsert→`delete_artifact_rows_except`→`delete_shadowed_table_rows`；失败自动回滚 |
| profile 联动 | `_refresh_metadata_documents_for_tables`（table_semantic_profile/store.py:112） | profile 删除/替换自动重建 metadata FTS 文档（kb_retrieval/store.py:817） |
| 按需补齐语义模型 | `auto_create.ensure_semantic_models_exist`（auto_create.py:377） | 设计上对 SQL 引用但缺语义模型的表自动生成；目前代码里无直接接线调用者（实际由 gen 节点在 OSI/指标流程内完成） |

### 需要手动兜底的边界（自动不覆盖）

- **表从库里删除/改名**：metadata 与语义模型旧向量行**不会自动清理**，`store_tables` 只增改不删，必须手动 `overwrite` 重建。
- **直接用编辑器改 YAML**：不自动同步，需手动 bootstrap 或走 CLI/agent 工具改。
- **reference SQL 内容变更**：incremental 下新 md5 会 append、旧行残留（不自动删），只有 overwrite truncate 才清。

一句话总结：**写侧（embed、生成、后台刷新、索引、同工件去重）基本全自动；删侧和源文件外部编辑是手动**——这正是设计上的一致性缺口。

---

## 七、项目复杂度 / 语义模型覆盖 / 检索准确性 / NL2SQL 准确性评估

基于代码与文档里的可核实证据（不是主观猜测）。

### 7.1 项目复杂度：很高，属于生产级规模

| 指标 | 数值 |
|---|---|
| 源码 | 602 个 `.py`，约 **21 万行**（datus/） |
| 测试 | 692 个测试文件，约 **27.8 万行** |
| 提交历史 | 990 commits |
| 关键子模块 | cli 106 文件/5.6 万行、tools 103/4.4 万、agent 60/2.5 万、storage 88/2.4 万、api 69/1.6 万 |

复杂度来源：**双轨 agent 编排**（`Node` 工作流 + `AgenticNode` 自由循环）、**可插拔存储后端**（`VectorDatabase`/`RDB` 抽象 → LanceDB/SQLite）、**多入口**（CLI/TUI/API/MCP/gateway）、**多数据源方言**、**Benchmark 三套**（BIRD/Spider2/Semantic Layer，benchmark/scripts/README.md）。

### 7.2 语义模型覆盖：机制强、覆盖有上限（依赖 SQL 语料）

**强项**：
- 三种来源（success story CSV / adapter / 已有 YAML）+ 会话内**按需自动补建**（`ensure_semantic_models_exist`，auto_create.py:377）：只要 SQL 引用过、库里没有，就会自动生成——覆盖是**由使用驱动的**。
- 粒度细：表/列拆成 `kind=table/column` 逐行入库（id 如 `column:orders.amount`），检索时按维度/度量/标识符重组（semantic_model/store.py:588）。

**上限/短板**（代码注释里明说了 tradeoff，docs/design/03:122）：
- 覆盖 = **被 SQL 碰过的表**。冷启动时从没被查询的表没有语义模型，只能走「原生 SQL 兜底」。
- 语义模型需要**随 schema 演进人工维护**（docs/design/03:122），库里删表不自动级联清理（无 GC）。
- profile/MetricFlow 投影覆盖取决于 adapter 格式。

### 7.3 检索准确性：设计上多路兜底、工程上扎实，但没有自动化回归保护

**准确性设计**：
- 三路检索（向量/FTS/标量）+ hybrid RRF 融合 + 失败降级（base.py:610）；字段级 boost（name 3×）；`matching_rate` 随 reflect 轮次**自适应扩大候选**（fast 5→medium 10→slow 20，NL2SQL_ACCURACY.md §1.7）。
- 精确回退：`get_schema`/`search_tables` 等值精查、标识符变体处理（大小写/引号/多级限定名）。
- 没建 KB 时直接连库读系统表兜底（schema_linking_node.py 路径 B）——**检索通道不会断链**。

**短板**（文档自曝）：
- docs/design/extract_knowledge_…:168：**"知识提取/检索准确性完全依赖 prompt 和模型质量，没有自动化回归保护"**。
- 向量检索的语义命中质量本质取决于 embedding 模型（默认 384 维小模型）与 `search_text` 的构造质量；`from_llm` 模式（LLM 选表）最准但最贵。

### 7.4 NL2SQL 准确性：机制密集，但仓库里没有可公开核对的数字

**准确性保障栈**（NL2SQL_ACCURACY.md「九大机制」）：
1. schema linking（RAG 注入表+列+样例值）→ 2. 生成即校验（SQLGlot + builtin_checks）→ 3. reflect 反思循环（三版可演进、失败分类）→ 4. 参考 SQL/模板检索复用历史成功 SQL（替代 few-shot）→ 5. BIRD/Spider2/Semantic Layer 量化。

**诚实结论**：
- 仓库里**没有公开的执行准确率百分比**（全仓 grep 无 "accuracy: xx%" 之类）。准确率只能通过 `uv run pytest -m nightly` + `benchmark` 在**真实 DB/带 API key** 环境跑出来，属于 nightly/regression 档，本地 CI 拿不到。
- 所以 NL2SQL 准确率**不是单一固定值**，而是随「模型 + 方言 + 知识库构建质量 + workflow 选型（fixed vs reflection）」浮动，设计上靠 Benchmark 度量（docs/design/03:329）。

**总体评价**：架构与工程完成度很高——检索与一致性机制完备、测试面大、有自省的 tradeoff 文档；但**准确性是"设计保障强、实证数字缺失"**：语义模型覆盖靠 SQL 语料驱动、检索/生成准确性靠 Benchmark 而非 CI 回归、删除一致性仍需手动兜底。

---

## 八、MetricFlow 与 OSI：定位 / 规格 / 编译器

### 8.1 结论

- **OSI（Open Semantic Interchange）= 编写规格（authoring spec）**：用户/LLM 维护的「源文件格式」。
- **MetricFlow = 执行后端 + 一套可选的编写格式**：它既是一种 YAML 规格（`data_source`/`measures`/`type_params`），也是一个真正把语义模型编译成 SQL 的引擎。
- **Dosi = 另一个执行后端**（Rust 原生引擎），和 OSI 用**同一套编写规格**。
- **编译器是有的**：`datus-semantic-osi` 把 OSI YAML 校验 → 编译成 Datus Semantic IR → 再 lower 到 MetricFlow；Dosi 则直接编译/规划/执行。

### 8.2 定位对比（docs/adapters/semantic_adapters.md）

| | 编写格式（source） | 执行后端 | 编译器路径 |
|---|---|---|---|
| MetricFlow 模式 | MetricFlow YAML（直接写） | MetricFlow | MetricFlow 自己的解析+IR+SQL 渲染 |
| OSI 模式 | **严格 OSI core YAML** + DATUS `custom_extensions` | 默认 MetricFlow（可换） | `datus-semantic-osi`：validate → compile **Datus Semantic IR** → lower 到 MetricFlow |
| Dosi 模式 | 同一套 OSI 格式 | **原生 Dosi（Rust）引擎** | OSI → 编译/join 规划/fan-out 保护/执行，不走 MetricFlow |

关键差异：**MetricFlow 模式把"后端字段"（`measure_proxy`、`type_params`、`data_source`）写进源文件；OSI 模式不让用户/LLM 碰这些后端字段**——它们由 OSI 适配器内部生成，是"一次性执行产物"，不是用户维护的源文件（docs/adapters/osi_semantic_adapter.md 明说）。

### 8.3 有没有编译器？

有，分成两段：
1. **datus-agent 仓库内**：`datus/api/utils/semantic_validation.py` 用 MetricFlow 库的 `ModelValidator`/`ConfigLinter`/`dir_to_model` 做深度校验（装了 `datus-metricflow` 时），否则降级为 YAML 语法检查。这只做**校验**，不做编译。
2. **独立仓库 `datus-semantic-adapter`**（通过 Python entry point `datus.semantic_adapters` 插件接入）：
   - `datus-semantic-core`：定义 OSI 核心 schema 与 **Datus Semantic IR**；
   - `datus-semantic-osi`：加载 OSI YAML → 校验 → **编译成 IR** → lower 到 MetricFlow；
   - `datus-semantic-metricflow`：MetricFlow 适配器（解析 YAML、列指标、取维度、查询/渲染 SQL、dry-run）；
   - `datus-semantic-dosi`：同一 OSI 源，直接进 Dosi 引擎编译执行。

Agent 侧（`gen_semantic_model`/`gen_metrics` 节点 + `osi-semantic-authoring` / `metricflow-semantic-authoring` 两个 skill）只负责**按当前激活的 adapter 写对应格式的 YAML**，编译执行全部交给 adapter 包，两段通过 `SemanticAdapterRegistry`（list_metrics / get_dimensions / query_metrics / validate_semantic）对接。

---

## 九、编写规格 / 执行后端 / 编译器在项目中的功能与流程

三个角色是**「源文件 → 编译产物 → 执行」三层**。

### 9.1 编写规格（Authoring Spec）：谁维护的源文件

**功能**：定义语义资产的人类/LLM 可读源格式。两种格式（`semantic_authoring.py:resolve_authoring_format` 按全局激活的 adapter 决定）：
- **MetricFlow YAML**：直接写 `data_source`/`measures`/`type_params` 等后端字段；
- **OSI core YAML + DATUS `custom_extensions`**：只写业务语义（dimensions/measures/identifiers/metrics + 业务提示），**不写任何后端字段**。

**在项目里的职责**：
- **生成**：`gen_semantic_model` / `gen_metrics` 节点按注入的 authoring skill（`osi-semantic-authoring` / `metricflow-semantic-authoring` / `osi-metrics-authoring`，见 semantic_authoring.py:40 的 `_REQUIRED_AUTHORING_SKILLS`）写 YAML 到 `subject/semantic_models/`、`subject/metrics/`。
- **权威源**：这些 YAML 是 source of truth；向量库只是它的投影（artifact replacement 保证一致性）。
- **校验**：发布前必须过 `validate_semantic`。

### 9.2 编译器（Compiler）：把源格式变可执行

**功能**：读源 YAML → 校验 schema → 编译成 IR → 降级到后端。三个 adapter 包（独立仓库 `datus-semantic-adapter`，通过 entry point `datus.semantic_adapters` 插件接入 `SemanticAdapterRegistry`）：

| 适配器 | 编译路径 |
|---|---|
| `datus-semantic-osi` | OSI YAML → 校验 → **Datus Semantic IR** → lower 到 MetricFlow |
| `datus-semantic-metricflow` | MetricFlow YAML → 直接解析 |
| `datus-semantic-dosi` | 同一 OSI 源 → 直接编译进 Rust 引擎 |

**在项目里的职责**：`datus/tools/semantic_tools/` 定义统一接口（`BaseSemanticAdapter`）：`list_metrics` / `get_dimensions` / `query_metrics(dry_run=True)` / `validate_semantic`。项目里另一个"半编译器"是 `datus/api/utils/semantic_validation.py`——装了 `datus-metricflow` 时用它的 `ModelValidator`/`ConfigLinter`/`dir_to_model` 做深度校验，否则降级 YAML 语法检查。**编译产物（MetricFlow 工件、SQL）是"一次性执行工件"，绝不当源文件入库**。

### 9.3 执行后端（Execution Backend）：真正产出 SQL 和数据

**功能**：把语义模型 + 指标 + 维度 → join 规划、聚合 → **渲染 SQL** → 连库执行返回结果。
- **MetricFlow**：渲染 SQL + 执行（OSI 的默认后端）；
- **Dosi**：原生 Rust 引擎做 join 规划 / fan-out 保护 / 执行，不依赖 MetricFlow。

**在项目里的职责**：agent 工具经 adapter 调它——`query_metrics`（semantic_tools.py:953）、归因分析（attribution_utils.py:179）、`ask_metrics` 工作流。

### 9.4 端到端流程（一条完整链路）

```
gen_semantic_model / gen_metrics 节点
   │  ① 编写规格：按 skill 写 OSI/MetricFlow YAML → subject/semantic_models/
   ▼
validate_semantic ──► 项目内 MetricFlow 深校验（或 YAML 语法兜底）
   │
   ▼
query_metrics(dry_run=True) ──► 编译器：OSI→IR→lower→MetricFlow（或 Dosi 直编）
   │                              渲染出可执行 SQL —— 这是发布前的 "可查询性门禁"
   ▼
publish / _sync_osi_semantic_to_db / _sync_semantic_to_db
   │  ② 编译+同步：语义对象/指标/表 profile 投影进向量库（KB）
   ▼
ask_metrics / search_metrics 用户问"上月营收按地区"
   │  ③ 查询：检索指标 → 经 adapter 调执行后端
   ▼
query_metrics ──► 编译 → SQL → 连库执行 → 返回数据（attribution 等再归因）
```

三个关键门禁（generation_tools.py:591-852）：**validate_semantic 必须通过 → 每个指标 query_metrics(dry_run=True) 必须通过 → 且必须按源 SQL 的 group-by 维度可查询**——全部通过才允许 publish，否则 LLM 拿错误信息自纠重试。

**一句话总结**：编写规格是"你/LLM 维护的源"；编译器是"把源降级成后端工件（含 SQL）"；执行后端是"拿工件渲染 SQL 并连库跑"。项目本体只做生成、校验、门禁、入库投影与查询编排，编译与执行全部委托给 adapter 插件包。

---

## 附：关键文件索引

| 主题 | 文件 |
|---|---|
| Embedding 模型 | `datus/storage/embedding_models.py` |
| 存储基类 | `datus/storage/base.py` |
| LanceDB 后端 | `datus/storage/vector/lance_backend.py` |
| 元数据存储/检索 | `datus/storage/schema_metadata/store.py`、`local_init.py` |
| 语义模型存储 | `datus/storage/semantic_model/store.py`、`semantic_model_init.py`、`auto_create.py`、`artifact_file.py` |
| Reference SQL | `datus/storage/reference_sql/store.py`、`reference_sql_init.py`、`sql_file_processor.py` |
| Reference Template | `datus/storage/reference_template/store.py`、`template_file_processor.py` |
| Subject 树 | `datus/storage/subject_tree/store.py` |
| FTS 元数据 RAG | `datus/storage/kb_retrieval/store.py` |
| 后台同步 | `datus/cli/background_sync.py` |
| 生成物同步钩子 | `datus/cli/generation_hooks.py` |
| Artifact 一致性 | `datus/storage/artifact_replacement.py` |
| KB 构建编排 | `datus/agent/agent.py`（`bootstrap_kb`） |
| 语义层适配器 | `datus/tools/semantic_tools/`、`docs/adapters/semantic_adapters.md` |
| Authoring 格式解析 | `datus/agent/node/semantic_authoring.py` |
| 语义校验 | `datus/api/utils/semantic_validation.py` |
