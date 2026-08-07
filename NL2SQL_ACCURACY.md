# Datus-Agent NL2SQL 准确性与质量保障深度分析

> 涵盖四轮完整对话：
> 1. Schema Linking 实现原理
> 2. NL2SQL 准确性提升的九大机制
> 3. Reflect Node 的判断可靠性与失败模式
> 4. 静默逻辑错误的处理现状与局限

---

## 目录

- [一、Schema Linking 是什么？如何实现？](#一schema-linking-是什么如何实现)
- [二、提升 NL2SQL 准确性的九大机制（按优先级）](#二提升-nl2sql-准确性的九大机制按优先级)
- [三、Reflect Node 如何判断结果对错？会误判吗？](#三reflect-node-如何判断结果对错会误判吗)
- [四、静默逻辑错误如何处理？](#四静默逻辑错误如何处理)

---

## 一、Schema Linking 是什么？如何实现？

### 1.1 定义

在 Text-to-SQL 场景中，用户说自然语言（"查一下加州学校的 SAT 成绩"），LLM 需要知道：

1. 这句话对应哪些**表**？（`satscores`、`schools`）
2. 每个表有哪些**列**？（`satscores.school_id`、`satscores.avg_scr_math`）
3. 列里有什么**示例值**？（`school_id = "1916676"`、`avg_scr_math = "450"`）

**Schema Linking 就是把自然语言查询"链接"到数据库 schema 的过程**——它是 Text-to-SQL 流水线的第一步，没有它 LLM 就不知道数据库里有什么。

输入/输出：

```
输入: 用户查询 + 数据库连接 → 输出: 相关的表名、列名、列类型、示例值
"查加州学校的数学成绩"    → [{table: satscores, columns: [school_id, avg_scr_math, ...]},
                             {table: schools, columns: [school_name, county, ...]}]
```

### 1.2 完整调用链

```
WorkflowRunner
  │
  └─ schema_linking_node.run()
       │
       └─ SchemaLinkingNode.execute()
            │
            └─ _execute_schema_linking()
                 │
                 ├─ 路径 A: RAG 知识库存在 → SchemaLineageTool.execute()
                 │    │
                 │    ├─ matching_rate="fast"    → _search_similar_schemas(top_n=5)
                 │    ├─ matching_rate="medium"  → _search_similar_schemas(top_n=10)
                 │    ├─ matching_rate="slow"    → _search_similar_schemas(top_n=20)
                 │    └─ matching_rate="from_llm"→ MatchSchemaTool (LLM 选表)
                 │         │
                 │         └─ storage.search_similar(query_text, top_n)
                 │              │
                 │              ├─ KbSearchMode.VECTOR → LanceDB 向量相似度搜索
                 │              └─ KbSearchMode.FTS    → SQLite FTS5 全文搜索
                 │
                 └─ 路径 B: RAG 知识库不存在 → _execute_schema_linking_fallback()
                      │
                      └─ tool.get_schems_by_db(connector)
                           └─ 直连数据库读 INFORMATION_SCHEMA / PRAGMA table_info
```

### 1.3 核心文件

**SchemaLinkingNode** — `datus/agent/node/schema_linking_node.py` (211 行)

注意：`SchemaLinkingNode` 继承的是 `Node`，**不是 `AgenticNode`**。它是少数不经过 LLM 推理循环的节点之一——它是一个纯搜索/查询节点。

```python
class SchemaLinkingNode(Node):
    def __init__(self, node_id, description, node_type, input_data, agent_config):
        super().__init__(...)
        self._table_schemas: List[TableSchema] = []
        self._table_values: List[TableValue] = []
```

### 1.4 核心调度：`_execute_schema_linking()`

```python
def _execute_schema_linking(self) -> SchemaLinkingResult:
    # 缓存检查：如果已有缓存结果直接返回
    if self._table_schemas:
        return SchemaLinkingResult(table_schemas=self._table_schemas, ...)

    # 路径 A: RAG 知识库已构建 → 向量/FTS 搜索
    if os.path.exists(self.agent_config.rag_storage_path()):
        tool = SchemaLineageTool(agent_config=self.agent_config)
        result = tool.execute(self.input, self.model)
        if result.success and len(result.table_schemas) > 0:
            return result
        return self._execute_schema_linking_fallback(tool)

    # 路径 B: RAG 知识库不存在 → 直连数据库读取
    else:
        return self._execute_schema_linking_fallback(SchemaLineageTool(...))
```

### 1.5 Fallback：直连数据库读取

```python
def _execute_schema_linking_fallback(self, tool):
    db_manager = db_manager_instance(self.agent_config.datasource_configs)
    connector = db_manager.get_conn(current_datasource, database_name)
    # 直接读数据库的系统表 (INFORMATION_SCHEMA / PRAGMA table_info)
    return tool.get_schems_by_db(connector=connector, input_param=self.input)
```

### 1.6 四种匹配率的渐进式搜索

**文件**: `datus/schemas/schema_linking_node_models.py`

```python
class SchemaLinkingInput(BaseInput):
    matching_rate: Literal["fast", "medium", "slow", "from_llm"] = "fast"

    def top_n_by_rate(self) -> int:
        if self.matching_rate == "fast":   return 5
        elif self.matching_rate == "medium": return 10
        return 20
```

| `matching_rate` | `top_n` | 行为 |
|---|---|---|
| `fast` | 5 | 快速返回 5 个最相关表 |
| `medium` | 10 | 中等返回 10 个候选 |
| `slow` | 20 | 最全面的搜索 |
| `from_llm` | N/A | 先召回候选 → LLM 从所有候选中挑选最相关的（MatchSchemaTool） |

### 1.7 与反思联动的自适应扩大

```python
# schema_linking_node.py:64-93
def setup_input(self, workflow):
    matching_rate = self.agent_config.schema_linking_rate  # "fast"
    matching_rates = ["fast", "medium", "slow", "from_llm"]
    start = matching_rates.index(matching_rate)
    # 关键: workflow.reflection_round 递增 → matching_rate 自动升级!
    # round 0 = fast(5), round 1 = medium(10), round 2 = slow(20)
    final_matching_rate = matching_rates[
        min(start + workflow.reflection_round, len(matching_rates) - 1)
    ]
```

如果 reflect 节点判定结果不好、回退到 schema_linking 重试，第二次就会搜索更多候选表。

### 1.8 底层搜索引擎：两种模式

**文件**: `datus/storage/kb_retrieval/store.py`

```python
class KbSearchMode(StrEnum):
    VECTOR = "vector"      # LanceDB 向量相似度搜索（默认）
    FTS = "fts"            # SQLite FTS5 全文搜索（更快，但需预构建）
```

**VECTOR 模式**：用户查询 → 嵌入向量 → LanceDB 余弦相似度 → top_n 个最相关表

```
"查加州学校的数学成绩" → embedding → [0.12, 0.34, ...]
    ↓ 余弦相似度匹配
  satscores (0.92) → school_id, avg_scr_math, avg_scr_read, ...
  schools   (0.87) → school_name, county, district, ...
  frpm      (0.65) → free_meal_count, reduced_meal_count, ...
```

**FTS 模式**：预先在 SQLite 中为 `table_name`、`schema_name`、`database_name`、文本描述建 FTS5 索引，查询时关键词匹配。比向量搜索更快，但缺少语义理解。

### 1.9 `from_llm` 模式：`MatchSchemaTool`

**文件**: `datus/tools/llms_tools/match_schema.py`

当 `matching_rate="from_llm"` 时，不走向量搜索，而是让 **LLM 直接挑选**：

```python
class MatchSchemaTool(BaseTool):
    def execute(self, input_data):
        # ① 先取出数据库中所有表
        table_metadata = self.storage.search_all(database_name=input_data.database_name)
        all_tables = gen_all_table_dict(table_metadata)

        # ② 让 LLM 从所有表中挑选最相关的
        match_result = self.match_schema(input_data, table_metadata, all_tables)
        #    → 用 gen_prompt() 渲染系统提示词
        #    → 列出所有候选表+列
        #    → LLM 返回 JSON: {"tables": ["satscores", "schools"], "reason": "..."}

        # ③ 基于 LLM 选择，构造最终结果
        return self._process_match_result(...)
```

### 1.10 数据流：Schema Linking → 下游节点

```
① schema_linking_node.update_context():
     workflow.context.table_schemas = [TableSchema(satscores, ...), TableSchema(schools, ...)]
     workflow.context.table_values  = [TableValue(school_id="1916676", ...)]

② gen_sql_node.setup_input():
     gen_sql.input.schemas = workflow.context.table_schemas  ← 读到上面写入的 schema

③ gen_sql_node 的 system_prompt 渲染:
     "You have access to these tables:
      - satscores: school_id, avg_scr_math, avg_scr_read, ...
      - schools: school_name, county, district, ...
      Sample values: school_id='1916676', avg_scr_math='450'"
```

### 1.11 Schema Linking 在 workflow.yml 中的位置

```yaml
reflection:
  - schema_linking    # ← 第 1 步：找到相关表
  - gen_sql           # ← 第 2 步：基于 schema 生成 SQL
  - execute_sql       # ← 第 3 步：执行
  - reflect           # ← 第 4 步：反思（可能回退到 schema_linking 扩大搜索范围）
  - output

metric_to_sql:
  - schema_linking    # ← 先找表
  - search_metrics    # ← 再找指标定义
  - date_parser       # ← 解析日期
  - gen_sql           # ← 基于表+指标+日期 生成 SQL
  - execute_sql
  - output
```

### 1.12 一句话总结

**Schema Linking 是 Text-to-SQL 的"眼睛"**——它用向量搜索/全文搜索/LLM 三种模式，从数据库几百张表中找到与用户问题最相关的那几张，提取表结构、列类型和示例值，然后注入到 gen_sql 节点的系统提示词中。没有它，LLM 就不知道该查哪些表。

---

## 二、提升 NL2SQL 准确性的九大机制（按优先级）

### 优先级一：多源知识库 RAG（Schema + 示例值 + 参考 SQL + 指标）

**这是对准确性影响最大的机制。** LLM 的能力上限被它"知道什么"限制。

项目中构建了 **4 种互补的知识库**，通过 `bootstrap_kb` 命令初始化：

| 知识库 | 存储内容 | 对准确性的贡献 |
|---|---|---|
| **schema_metadata** | 表名、列名、列类型、列注释 → 向量 + FTS 索引 | 让 LLM 知道"有哪些表和列可用" |
| **reference_sql** | 黄金 SQL 样例 → 向量索引 | 让 LLM 参考"正确 SQL 长什么样"（few-shot） |
| **metrics** | 指标定义（名称、聚合逻辑、维度） | 让 LLM 知道"sum(amount) 叫 total_revenue" |
| **semantic_model** | 表间关系、外键、业务语义 | 让 LLM 知道"schools JOIN satscores ON school_id" |

**底层搜索**：LanceDB 向量相似度 + SQLite FTS5 全文搜索双模式。

---

### 优先级二：Schema Linking 渐进式搜索 + 反思联动

```
用户查询 → schema_linking(fast, top_n=5)
  → gen_sql → execute_sql → reflect
       │ 结果不对？回退 ↓
  schema_linking(medium, top_n=10)
  → gen_sql → execute_sql → reflect
       │ 还不对？再扩大 ↓
  schema_linking(slow, top_n=20) → ...
```

这是**搜索范围的自适应扩大**——第一次快速找 5 张表，如果 SQL 执行结果不好（reflect 判定），第二次找 10 张，第三次找 20 张，极端情况让 LLM 从全部候选中挑。

---

### 优先级三：Workflow 级反思回退（Reflect Node）

```yaml
reflection:
  - schema_linking
  - gen_sql
  - execute_sql
  - reflect      # ← 审视结果，决定是否回退
  - output
```

`reflect_node` 收到 SQL 执行结果后，用 LLM 判断是否需要回退。每次反思都带有语义理解——LLM 知道自己犯了什么错误，用错误信息作为新 prompt 重新生成。

---

### 优先级四：高信息密度提示词工程

**文件**: `datus/prompts/prompt_templates/gen_sql_system_*.j2` (60+ 模板文件)

`GenSQLAgenticNode` 在每次调用前通过 `prepare_template_context()` 注入大量动态上下文：

```python
context = {
    "has_db_tools": True,                    # 可用 → 告诉 LLM 可以执行 SQL
    "has_context_search_tools": True,         # 可用 → 可以搜索参考 SQL
    "has_parsing_tools": True,               # 可用 → 可以解析日期
    "has_semantic_tools": True,              # 可用 → 可以查指标定义
    "has_reference_template_tools": True,    # 可用 → 可以查模板
    "native_tools": ["execute_sql", "search_reference_sql", ...],
    "mcp_tools": [...],
    "scoped_context": True/False,            # 有限上下文模式
    "rules": ["优先使用 reference SQL 中的写法", ...],
    "agent_description": "你是一个 SQL 专家...",
    "datasource": "california_schools",
    "db_name": "california_schools",
}
```

**关键设计**：
- 模板**按版本管理**（如 `gen_sql_system_1.2.j2`），可以 A/B 测试不同提示词
- **每会话缓存**：同一版本不重复渲染
- **`scoped_context` 模式**：当用户指定了具体几张表时，提示词会强调"只用这几张表"
- **`rules` 规则注入**：可以在 `agent.yml` 中配置 SQL 编写规范

---

### 优先级五：参考 SQL 知识库（Few-Shot Learning）

项目中可以导入黄金 SQL 样例（`--sql_dir`），LLM 生成 SQL 前可以先搜索相似的参考 SQL：

```
用户: "计算每个学校的平均数学成绩"

LLM 调用 search_reference_sql("average math score per school")
  → 返回最相似的参考 SQL:
    "SELECT school_name, AVG(avg_scr_math) FROM schools
     JOIN satscores ON schools.school_id = satscores.school_id
     GROUP BY school_name"

LLM 看到参考 → 模仿其 JOIN 方式、列名、聚合模式
  → 生成对应的 SQL
```

**工具链**：
- `search_reference_sql(query)` — 向量搜索相似 SQL
- `get_reference_sql(id)` — 获取完整 SQL 详情
- `list_reference_sqls(database)` — 列出某数据库的所有参考 SQL

---

### 优先级六：Validation 双 Layer 校验

**文件**: `datus/validation/`

```
Layer A (builtin_checks.py): 确定性检查，不需要 LLM
  • TableTarget → describe_table，确认表确实被创建了
  • TransferTarget → 行数一致性检查
  • DashboardTarget → BI 工具确认仪表板存在
  • SchedulerJobTarget → 调度任务存在性 + 状态检查

Layer B (llm_runner.py): LLM-as-Judge 语义检查
  • SQL 逻辑是否合理？
  • 输出是否符合用户意图？
  • 边界条件是否处理？
```

**ValidationHook** 在每个增长型工具调用后自动触发。

---

### 优先级七：Scoped Context（有限上下文）

当用户指定了 `--tables frpm,satscores`，GenSQL 节点只把这些表的信息注入系统提示词，不加载整个数据库 schema。

```python
scoped_context = node_config.scoped_context
has_scoped_context = bool(
    scoped_context and (scoped_context.tables or scoped_context.metrics or scoped_context.sqls)
)
context["scoped_context"] = has_scoped_context
```

**效果**：减少提示词噪音、减少幻觉、减少 Token 消耗。

---

### 优先级八：日期解析（Date Parsing）

用户常说"上个月"、"去年 Q3"、"最近 30 天"，LLM 可能搞错时间范围。项目中专门有一个 `date_parser` 节点 + 工具：

```
用户: "上个月的销售额"

date_parser_node 解析:
  "上个月" → {
    "current_date": "2026-08-07",
    "resolved": {
      "start_date": "2026-07-01",
      "end_date": "2026-07-31",
      "date_column": "sale_date"
    }
  }

→ 注入到 gen_sql 的 user_prompt:
  "查询 2026-07-01 到 2026-07-31 之间的销售额"
```

**消除了 LLM 对时间表达式的歧义解读**——日期解析是确定性的。

---

### 优先级九：SQL Policy + Guard 安全守护

**文件**: `datus/tools/sql_guard.py` + `datus/tools/sql_policy.py`

```
LLM 生成: "DROP TABLE schools"
    ↓ sql_guard.check(sql): ✗ 拒绝

LLM 生成: "SELECT * FROM sales" (无 LIMIT, 表有 1000 万行)
    ↓ sql_policy.enforce(sql): ✗ 追加 LIMIT 1000
```

防止错误 SQL 造成数据灾难。

---

### 总结：优先级一览

```
优先级   机制                   原理                         失败的后果
─────── ────────────────────── ──────────────────────────── ──────────────────
  P1    多源 RAG 知识库         让 LLM 知道有什么表和列        生成不存在的表名/列名
  P2    渐进式 Schema Linking   搜索范围随反思轮次扩大         漏掉关键表
  P3    Workflow 反思回退       LLM 审视错误→带着反馈重试      一次失败就终止
  P4    高信息密度提示词         动态注入所有可用上下文         缺乏上下文生成低质量 SQL
  P5    参考 SQL 知识库         Few-shot 模仿黄金样例         语法正确但逻辑错误
  P6    Validation 双层校验     确认产出物真实可用             生成了不存在的表/仪表板
  P7    Scoped Context          缩小搜索空间，减少幻觉         被无关表干扰
  P8    日期解析                 确定性解析，不交给 LLM 猜     时间范围错误
  P9    SQL Policy + Guard      防破坏 + 追加 LIMIT           数据灾难/超时
```

**最核心的三件事**：① 给 LLM 足够且精准的上下文（P1+P2+P7）→ ② 让 LLM 能看到好样例（P5+P4）→ ③ 让 LLM 有机会自我纠正（P3+P6）。

---

## 三、Reflect Node 如何判断结果对错？会误判吗？

### 3.1 判断全链路

**第一步：`evaluate_with_model()` — 纯 LLM 判断**

**文件**: `datus/agent/reflect.py:16-69`

```python
def evaluate_with_model(task, node_input, model, agent_config):
    sql_context = node_input.sql_context[-1]

    evaluate_template = get_evaluation_prompt(
        task_description=task.to_str(),           # 用户原始问题
        sql_generation_result=sql_context.sql_query,  # LLM 生成的 SQL
        sql_execution_result=(                    # 数据库返回的执行结果
            f"SAMPLE ROWS: {sample_return}\n"
            f"ERROR: {sql_context.sql_error}\n"
            f"Rows_returned: {sql_context.row_count}"
        ),
    )

    # LLM 判定 → 强制 JSON 输出
    evaluation = model.generate_with_json_output(evaluate_template)
    classification = evaluation["classification"]
    return {"success": True, "strategy": classification, "details": evaluation}
```

**传给 LLM 的信息**：

| 信息 | 来源 | 可靠性 |
|---|---|---|
| 用户原始问题 | `task.task` | 可靠（用户提供） |
| 生成的 SQL | `sql_context.sql_query` | **不可靠**（LLM 生成的） |
| 执行错误信息 | `sql_context.sql_error` | 可靠（数据库返回的） |
| 返回行数 | `sql_context.row_count` | 可靠（数据库返回的） |
| 样本数据 | `sql_context.sql_return[:n]` | 可靠（数据库返回的） |

**第二步：LLM 看到的评估提示词**

**文件**: `datus/prompts/prompt_templates/evaluation_2.1.j2`

```
You are an expert SQL evaluator. Classify into one of:

1. SUCCESS - Results are accurate, valid and ready for output.
   If no result found, you should carefully check the SQL or schema.

2. DOC_SEARCH - Unsure about SQL syntax/structure/dialect.
   Search documentation with specific keywords.

3. SIMPLE_REGENERATE - Minor syntax errors only (missing comma, typo).
   You must be highly confident about the dialect.

4. SCHEMA_LINKING - Schema mismatches. Re-run schema analysis.

5. REASONING - Complex issues. Need column exploration and data sampling.

Task: {{ task_description }}
Generated SQL: {{ sql_generation_result }}
Execution results: {{ sql_execution_result }}
```

**第三步：策略分发**

```python
def _execute_reflection_strategy(self, strategy, details, workflow):
    if strategy == "SUCCESS":
        return {"success": True, "message": "go on to output"}   # ← 直接通过！

    max_round = get_env_int("MAX_REFLECTION_ROUNDS", 3)
    if workflow.reflection_round > max_round:
        return {"success": True, "message": "Max reflection rounds exceeded"}

    if strategy in [DOC_SEARCH, SCHEMA_LINKING, SIMPLE_REGENERATE, REASONING]:
        return self._execute_strategy(details, workflow, strategy)
```

当 strategy = `SUCCESS` 时，**不做任何验证**，直接前进到 output 节点。

---

### 3.2 四个典型误判场景

#### 场景一：SQL 语法正确但逻辑错误

```
用户问题: "每个县的免费餐学生比例"

生成的 SQL:
  SELECT county, SUM(free_meal_count) / COUNT(*) FROM frpm GROUP BY county
  实际应该: SUM(free_meal_count) / SUM(total_enrollment)  -- 分母错了

执行结果:
  ✓ 5 rows returned, no error
  county     | ratio
  Alameda    | 0.34
  Los Angeles| 0.28

LLM 评估:
  → "5 rows, no error, ratios look plausible" → SUCCESS ✓
  ✗ 但实际上分母用错了，比例全算错了！
```

**为什么会错判**：LLM 没有真实数据作为 ground truth。0.34 看起来像一个合理的比例，但 LLM 不知道实际应该是 0.45。

#### 场景二：空结果被误判为数据不存在

```
用户问题: "查 Alameda 县 2025 年的 SAT 成绩"

生成的 SQL:
  SELECT * FROM satscores WHERE county = 'Alameda' AND year = 2025
  实际: satscores 表没有 county 列，需要 JOIN schools 表

执行结果:
  ✗ ERROR: no such column: county

LLM 评估:
  → 这是语法错误，会被正确判定为 SCHEMA_LINKING 或 SIMPLE_REGENERATE
```

这个场景实际上**能被检测**，因为有数据库硬错误信号。

#### 场景三：返回行数异常但 LLM 不敏感

```
用户问题: "查询学校总数"

生成的 SQL (正确): SELECT COUNT(*) FROM schools
实际返回: 1 row: "count = 1500"

生成的 SQL (错误): SELECT COUNT(*) FROM schools s JOIN frpm f ON ...
实际返回: 1 row: "count = 1500"  -- 看起来一样！

LLM 评估:
  → "count returned correctly" → SUCCESS
  ✗ 但用了不必要的 JOIN，且可能因重复而变化
```

#### 场景四：类型转换静默失败

```
用户问题: "平均数学成绩 > 500 的学校"

生成的 SQL:
  SELECT school_name FROM schools
  WHERE school_id IN (
    SELECT school_id FROM satscores WHERE avg_scr_math > '500')
  -- avg_scr_math 是 TEXT 类型, '500' > '1000' 在字符串比较中为 True!

执行结果:
  ✓ 返回了 200 行
  ✗ 但字符串比较导致结果完全错误

LLM 评估:
  → "200 rows returned, looks correct" → SUCCESS
  ✗ 它不知道 avg_scr_math 的类型，无法发现类型陷阱
```

---

### 3.3 可靠性边界

```
┌───────────────────────────────────────────────────────────────┐
│                  Reflect Node 的可靠性边界                      │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│  可靠判断:                                                    │
│  ✓ 数据库返回了 ERROR → "SQL 有问题，修正"                     │
│  ✓ 返回 0 行 + schema 明显不匹配 → "需要重新搜 schema"         │
│  ✓ 返回行数异常大(如 1000 万) → "需要限定范围"                 │
│                                                               │
│  不可靠判断:                                                   │
│  ✗ SQL 语法正确、执行成功、返回数据、但逻辑错误                │
│  ✗ 类型隐式转换导致的静默错误                                  │
│  ✗ JOIN 了错误/多余的表但返回了数据                            │
│  ✗ 聚合方式不对但结果看起来 plausible                          │
│                                                               │
│  根本原因: LLM 没有 ground truth。它只能通过:                   │
│  - 执行错误（硬信号）→ 可靠                                    │
│  - 输出合理性（软信号）→ 不可靠                                │
│  - 行数与预期的匹配度（软信号）→ 不可靠                        │
│                                                               │
│  防御深度:                                                    │
│  Layer 1: DB 错误返回 (硬)   ← 可靠                            │
│  Layer 2: LLM 语义判断 (软)  ← 本项目唯一的一层                 │
│  Layer 3: evaluate.py 管道  ← 只检查数据传递                   │
│  Layer 4: Validation        ← 只检查产出物存在性                │
│                                                               │
│  没有的防线:                                                  │
│  ✗ 执行结果与预期值的数值比较（没有 ground truth）              │
│  ✗ 结果可复现性检查（不会用不同参数再查一遍验证）              │
│  ✗ 人类审查环节（除非用户主动要求）                            │
└───────────────────────────────────────────────────────────────┘
```

**一句话结论**：Reflect Node 对**语法错误和明显的 schema 不匹配**可靠，对**静默的逻辑错误**不可靠。当 LLM 判断 `SUCCESS` 时，它判断的不是"SQL 逻辑正确"，而是"执行没报错 + 返回了数据 + 数据看起来合理"。这是一个概率性判断，**确实可能把错误的当对的**。

---

## 四、静默逻辑错误如何处理？

### 4.1 核心发现：几乎没有可靠的检测机制

项目中能查到的每一个相关机制：

### 机制一：`OutputTool.check_sql()` — 但仅在 benchmark 模式启用

**文件**: `datus/tools/output_tools/output.py:98-161`

```python
def check_sql(self, input_data, sql_connector, model):
    if not input_data.check_result:      # ← 默认 False！只有 benchmark 才开
        return input_data.gen_sql, input_data.sql_result

    prompt = gen_prompt(
        user_question=input_data.task,
        table_schemas=input_data.table_schemas,
        sql_query=input_data.gen_sql,             # LLM 生成的 SQL
        sql_execution_result=input_data.sql_result, # 执行结果
        metrics=input_data.metrics,
    )
    llm_result = model.generate_with_json_output(prompt)

    if llm_result.get("is_correct", True):        # LLM 判断 SQL 对不对 → 又是 LLM 判断 LLM
        return input_data.gen_sql, input_data.sql_result

    final_sql = llm_result.get("revised_sql")
    final_result = sql_connector.execute({"sql_query": final_sql})
    return final_sql, final_result
```

跟 reflect_node 一样的问题——**又是 LLM 判断 LLM**。同一个模型同时扮演"考生"和"阅卷老师"。

### 机制二：Reflect Node — LLM 自裁判

已分析过。对于静默错误，不可靠。

### 机制三：output_checking 提示词模板 — 又一个 LLM 判断

**文件**: `datus/prompts/output_checking.py:13-75`

给 LLM 的输入和 reflect node 几乎一样：用户问题 + 生成的 SQL + 执行结果 + schema。用的裁判是同一个 LLM。

### 机制四：MAX_REFLECTION_ROUNDS=3 — 投降机制

```python
if workflow.reflection_round > max_round:
    return {"success": True, "message": "Max reflection rounds exceeded"}
```

3 轮没修好 → 强行通过。

---

### 4.2 缺失的防线（诚实列表）

| 缺失的防线 | 为什么重要 | 为什么没有 |
|---|---|---|
| **执行结果交叉验证** | 用不同的 SQL 写法查同一个问题，比较结果是否一致 | 成本高（2x+ LLM token），且不一定能生成语义等价的替代 SQL |
| **结果数值合理性检查** | "平均数学成绩 = 450" 是否在合理范围（0-800） | 需要预定义列的取值范围，需要人工标注 |
| **与参考 SQL 结果比对** | 同一个问题有没有黄金 SQL？比对结果 | 只有 benchmark 有标准答案，生产环境没有 |
| **结果分布检查** | 返回了 5 行，是不是应该有 50 个县？ | 不知道正确答案是什么 |
| **行数预期检查** | "每个学校的平均分"——返回 1500 行，是1500所学校吗？ | 没有元数据告诉系统"应该有1500所学校" |
| **用户确认环节** | 让用户看一眼结果是否合理 | 自动化流程不能中断 |

---

### 4.3 完全诚实的结论

```
┌─────────────────────────────────────────────────────────────────┐
│           静默逻辑错误的处理能力矩阵                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  错误类型                        能否检测      检测方式           │
│  ─────────────────────────────────────────────────────────────  │
│  语法错误 (SELEC → SELECT)        ✓ 能         DB 报错 (硬)      │
│  表不存在 (FROM sales)            ✓ 能         DB 报错 (硬)      │
│  列不存在 (SELECT foo FROM ...)   ✓ 能         DB 报错 (硬)      │
│  类型不匹配 (WHERE id = 'abc')    ✓ 能         DB 报错 (硬)      │
│  ─────────────────────────────────────────────────────────────  │
│  JOIN 了错误的表                  ✗ 不能       LLM 猜 (不可靠)    │
│  聚合方式错误 (SUM vs AVG)        ✗ 不能       LLM 猜 (不可靠)    │
│  WHERE 条件遗漏/多余              ✗ 不能       LLM 猜 (不可靠)    │
│  子查询逻辑错误                   ✗ 不能       LLM 猜 (不可靠)    │
│  日期范围算错                     ✗ 不能       LLM 猜 (不可靠)    │
│  字符串比较代替数值比较           ✗ 不能       LLM 猜 (不可靠)    │
│  NULL 处理不当                    ✗ 不能       LLM 猜 (不可靠)    │
│  GROUP BY 缺少必要列              ✗ 依赖       DB 可能报错        │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  根本原因:                                                       │
│  项目没有任何 ground truth 比对机制。                              │
│  所有的"质量检查"本质上是 LLM 审视 LLM 的输出。                    │
│                                                                 │
│  唯一的真实验证发生在 benchmark 模式:                              │
│  生成的 SQL 的结果 vs 标准答案的 SQL 的结果 → 准确率数字           │
│  但这是离线评估，不在生产流程中。                                   │
│                                                                 │
│  设计哲学:                                                       │
│  "把上下文给足 (RAG + schema + reference SQL)，                   │
│   让 LLM 一次写对，                                               │
│   如果写错了 → reflect 再试一次，                                 │
│   如果 reflect 也判断错了 → 接受这个结果。"                         │
│                                                                 │
│  这不是一个 bug，这是当前 NL2SQL 技术栈的固有限制。                  │
│  项目的策略是"预防"而非"检测"——通过充分的上下文                      │
│  降低出错概率，而不是在出错后检测并修复。                             │
└─────────────────────────────────────────────────────────────────┘
```

**一句话总结**：项目对静默逻辑错误**没有可靠的检测机制**。它依赖的是"给 LLM 足够上下文 → 一次写对"的预防策略，加上 reflect node 的"LLM 自己再看一遍"的补救策略。两者都是概率性的。唯一的真实验证发生在离线 benchmark 评估中（对比标准答案），不在生产流水线中。

---

## 附录：相关文件索引

| 文件 | 行数 | 对应章节 |
|---|---|---|
| `datus/agent/node/schema_linking_node.py` | 211 | 一 |
| `datus/tools/lineage_graph_tools/schema_lineage.py` | ~150 | 一 |
| `datus/schemas/schema_linking_node_models.py` | 98 | 一 |
| `datus/tools/llms_tools/match_schema.py` | ~200 | 一 |
| `datus/storage/kb_retrieval/store.py` | ~550 | 一 |
| `datus/agent/node/gen_sql_agentic_node.py` | 945 | 二 |
| `datus/agent/node/reflect_node.py` | 223 | 三 |
| `datus/agent/reflect.py` | 70 | 三 |
| `datus/prompts/prompt_templates/evaluation_2.1.j2` | 30 | 三 |
| `datus/prompts/reflection.py` | 34 | 三 |
| `datus/tools/output_tools/output.py` | 215 | 四 |
| `datus/prompts/output_checking.py` | 76 | 四 |
| `datus/validation/builtin_checks.py` | ~250 | 二 |
| `datus/agent/node/deliverable_node.py` | ~400 | 二 |
| `datus/agent/node/retry_policy.py` | 80 | 二 |
| `datus/tools/sql_guard.py` + `sql_policy.py` | — | 二 |
| `datus/tools/func_tool/date_parsing_tools.py` | — | 二 |
