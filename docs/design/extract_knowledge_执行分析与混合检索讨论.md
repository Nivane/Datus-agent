# extract-knowledge 技能执行分析与混合检索讨论

> 对话记录：从技能执行链路分析到 RRF、混合检索的延伸讨论

---

## 一、extract-knowledge 技能执行全景

### 核心发现

`extract_knowledge` 在 `datus/` 中**没有任何 Python 实现代码**。它是一个纯 prompt-driven 技能：agent 读取 `SKILL.md` 中的指令，通过调用已有工具（`execute_sql`、`task`、`read_file`、`write_file`、`edit_file`、`ask_user`）按步骤执行。

**历史背景**：`extract_knowledge` 曾经有 Python 实现（`datus/storage/ext_knowledge/`），在 PR #932 中被移除并替换为当前的纯 prompt-skill 方案。CI 配置 `ci/run-pr-tests.py:120` 还残留了对已删除目录的映射引用（stale reference）。

### 加载机制链路

```
skill_manager.load_skill("extract-knowledge")
  → skill_registry._scan() 发现 SKILL.md（rglob 扫描 datus/resources/skills/）
  → skill_config.SkillMetadata 解析 frontmatter（user_invocable: true, disable_model_invocation: false）
  → skill_func_tool.SkillFuncTool.load_skill() 暴露为模型可调用的 function tool
  → SKILL.md 内容注入 agent 上下文，agent 按指令逐步执行
```

---

### 触发路径

| 触发来源 | 触发时机 | 调用方式 |
|----------|----------|----------|
| **`/init`** | Step 2 — 扫描表元数据/文档时发现原子业务事实 | **lite** 模式 |
| **`/build-kb`** | Step 3 — dual-route：同一个 `(question, SQL)` 既送 `gen_sql_summary` 也送 `extract-knowledge` | **lite** 模式 |
| **`session-summarize`** | Step 3 — 分类为 `knowledge` 的候选条目 | **lite** 模式 |
| **用户直接调用** | `/extract-knowledge`，从消息中解析 pair 或从对话上下文恢复 | 用户选 lite/deep |
| **自动触发** | 查询成功后推理过程揭示了不可推断的规则（`disable_model_invocation: false`） | 模型自行判断 |

---

### Lite 模式执行流程

```
Step 1: execute_sql(gold_sql) → 验证 gold SQL 能跑，缓存结果
    ↓ 失败 → 跳过该 pair

Step 2-4(lite): 主 agent "假装不知道 gold SQL"
    → 仅凭 question + schema 写一份草稿 SQL (不执行)
    → 沿 8 个维度 diff 草稿 vs gold_sql:
       表选择 · join 类型/键 · WHERE 常量过滤 · GROUP BY 粒度
       · 聚合/去重 · 输出列 · 边界条件 · 术语→字段映射
    → 每个 diff 项都是候选 fact

Step 5: 系统性语料扫描 + "worth-writing" 四问过滤

Step 6: 持久化
    → 6.1 解析目标 domain 文件 (优先复用已有 domain)
    → 6.2 语义级去重/冲突检测 (duplicate/refinement/conflict/complementary/derivable)
    → 6.3 write_file 或 edit_file 写入 ./knowledge/<domain-slug>.md

Step 7: 更新 AGENTS.md ## Knowledge 索引入口
```

### Deep 模式执行流程（严格校验，有 subagent）

```
Step 1: execute_sql(gold_sql) → 验证

Step 2: task(type="gen_sql", prompt=<仅question>)
    → gen_sql subagent 在盲态下生成 SQL（从未见过 gold_sql）
    → 保存 session_id

Step 3: execute_sql(候选SQL) → 与 gold 结果比对
    → row count / columns / 排序后采样 / EXCEPT 差异探测
    → 判决: match 或 mismatch

Step 4 (mismatch): 诊断 → 重试（最多 5 轮）
    → 仅描述症状和方向（不泄露 gold 值）
    → task(type="gen_sql", session_id=上轮id, prompt=<hint>)
    → 循环 Step 3→4

Step 5: 对 gen_sql 的最终 SQL vs gold_sql 做 diff
    （若 5 轮后仍 mismatch → 写入已知 gap 到 Open Gaps）

Step 6 & 7: 同 lite 的持久化 + 索引更新
```

### Deep 模式的 gen_sql subagent

通过 `sub_agent_task_tool.py` 的 `task()` 工具，`type="gen_sql"` 映射到 `GenSQLAgenticNode`（`datus/agent/node/gen_sql_agentic_node.py`）。

---

### 涉及的工具调用

| 工具 | Lite | Deep | 用途 |
|------|------|------|------|
| `execute_sql` | ✓ | ✓ | 验证 gold SQL + 执行 subagent SQL + 差异探测 |
| `task(type="gen_sql")` | ✗ | ✓ | 盲态委托 SQL 生成；`session_id` 跨轮复用上下文 |
| `read_file` | ✓ | ✓ | 持久化前读取 `./knowledge/*.md` 做去重/冲突检测 |
| `write_file` | ✓ | ✓ | 新建 domain 文件 |
| `edit_file` | ✓ | ✓ | 向已有 domain 文件追加/修改事实 |
| `ask_user` | ✓ | ✓ | 输入确认、模式选择、新 domain 确认、冲突决策 |

### 产物与副作用

| 产物 | 路径 | 格式 |
|------|------|------|
| 知识文件 | `./knowledge/<domain-slug>.md` | Markdown，按 domain → topic → facts 层级 |
| 索引更新 | `./AGENTS.md` 的 `## Knowledge` 节 | 每个 domain 一行链接 + scope |
| Open Gaps（仅 deep） | `AGENTS.md → ### Open Gaps` | 5 轮耗尽仍未对齐的差异 |

---

### "Worth-Writing" 四问过滤

任一答"是"→丢弃：

1. 没有这条知识，有 schema + question 的 SQL agent 能否答对？
2. 能否从 `INFORMATION_SCHEMA` / 表注释 / 列名直接推断？
3. 这是通用 SQL 知识（非业务/数据集特有）？
4. 该事实能否从已有 facts 机械组合得出？

---

## 二、设计 Gap：extract-knowledge 不更新其他存储

### 问题

`extract-knowledge` 的 SKILL.md 里完全不涉及以下四个存储：

| 存储 | 写入机制 | extract-knowledge 是否更新 |
|------|----------|--------------------------|
| `knowledge` | `write_file`/`edit_file` → `./knowledge/*.md` | ✓ 唯一产物 |
| `reference_sql` | `task(gen_sql_summary)` → `./subject/sql_summaries/` + LanceDB | ✗ |
| `semantic_models` | `task(gen_semantic_model)` → `./subject/semantic_models/` + LanceDB | ✗ |
| `metrics` | `task(gen_metrics)` → LanceDB `metrics` 表 | ✗ |
| `reference_template` | `reference_template_tools.py` → `./subject/reference_templates/` | ✗ |

### 影响

`extract-knowledge` 是单一存储写入器。`/build-kb` 和 `session-summarize` 通过 `storage-classify` 的 dual-route 完成多存储写入，但 `/extract-knowledge` 独立运行时不做：

1. 知识原子被提取了（如 `status='A' = 活跃`）
2. 但这条 SQL 本身没有被索引为 `reference_sql`
3. 没有生成 `semantic_model`
4. 没有生成 `metrics`
5. 没有生成 `reference_template`

`storage-classify` 的 disambiguation 明确写了 dual-route 的必要性：

> A single (question, gold_sql) pair routes to BOTH: store the example, and run extract-knowledge to mine the rule.

但 `/extract-knowledge` 独立运行时没有实现这个 dual-route。这是设计 gap。

---

## 三、测试覆盖

没有针对 `extract-knowledge` 技能的专项测试。现有的 skill 机制测试仅覆盖加载路径：

- `tests/unit_tests/tools/skill_tools/test_skill_config.py`
- `tests/unit_tests/tools/skill_tools/test_skill_registry_unit.py`
- `tests/unit_tests/tools/skill_tools/test_skill_manager_unit.py`
- `tests/unit_tests/tools/skill_tools/test_skill_func_tool.py`
- `tests/integration/tools/test_skill.py`
- `tests/integration/tools/test_skill_execution.py`
- `tests/unit_tests/cli/test_skill_commands.py`

[INFERRED] 知识提取的准确性完全依赖 prompt 质量和模型能力，没有自动化回归保护。

---

## 四、延伸讨论：Reciprocal Rank Fusion (RRF)

### 定义

**倒数排名融合**——用倒数来融合不同来源的排名。

### 公式

```
RRF_score(d) = Σ 1 / (k + rank_i(d))
```

- `d` — 候选文档/结果
- `rank_i(d)` — 文档在检索源 `i` 的排名（从 1 开始）
- `k` — 平滑常数，经典值 60

### 为什么需要

多路召回场景中，不同检索源的打分分布不同（向量 cosine 距离 [0, 2]；BM25 可能是 [0, 50+]），无法直接比大小。RRF 绕过归一化——只关心**排名**，不关心绝对分数。

### k 的作用

`k` 是分母中的平滑常数，防止第一名权重过冲：

| 排名 | 无 k `1/rank` | k=60 `1/(60+rank)` |
|------|--------------|--------------------|
| 1 | 1.00 | 0.0164 |
| 2 | 0.50 | 0.0161 |
| 3 | 0.33 | 0.0159 |
| 10 | 0.10 | 0.0143 |
| 100 | 0.01 | 0.0063 |

无 k 时排名 1 的分是排名 2 的 2 倍，k=60 把差距压到 2%。融合结果由"多个来源的共识"驱动，不被某个来源的第一名绑架。

来源：Cormack et al., *Reciprocal Rank Fusion outperforms Condorcet and individual rank learning methods*, SIGIR 2009。

### 融合步骤

1. 每个检索源返回排序列表
2. 标注每个结果在各来源的排名（从 1 开始）
3. 对每个文档计算 `1/(k + rank)`，跨来源求和
4. 按总分降序排列，取 top-N

```python
def rrf(results_per_source: list[list[str]], k: int = 60) -> list[str]:
    scores = {}
    for source_results in results_per_source:
        for rank, doc_id in enumerate(source_results, start=1):
            scores[doc_id] = scores.get(doc_id, 0) + 1.0 / (k + rank)
    return sorted(scores.keys(), key=lambda d: scores[d], reverse=True)
```

### 关键取舍

- **不用调参**：k=60 开箱即用，排名结果对 k 不敏感
- **丢失分数信息**：只按排名算，忽略排名的实际分数差距——这是"不需要归一化"的代价
- **不处理重复**：同一结果出现在多个来源时自动加权，但要考虑是否设排名截断

---

## 五、延伸讨论：混合检索 vs 多源联合检索

### 混合检索（Hybrid Search）— 业界标准定义

**同一个语料库上，两种不同检索算法的组合**——几乎专指 dense（向量）+ sparse（关键词/BM25）：

```
同一个 reference_sql 库
  ├── 向量检索: embedding 匹配语义
  └── BM25 检索: 关键词精确匹配
        ↓ RRF 融合
     最终结果
```

Elasticsearch、Pinecone、Weaviate、Milvus、Vespa 等系统中说的 "hybrid search" 指的就是这个。

### 多源联合检索（Multi-Source Federation）

**同一个查询打到不同的知识库上**，各自返回结果再合并：

```
question →  search_semantic_model → top-k
         →  search_metrics       → top-k
         →  search_reference_sql → top-k
              ↓ 合并去重
           最终 prompt 上下文
```

这不是 hybrid search，是 multi-source retrieval 或 federated search。

### Datus-Agent 的实际情况

`context_search` 做的是**多源联合检索**——同一个问题同时路由到 semantic_model、metrics、reference_sql 三个独立的 LanceDB 向量 store，分别查 top-k，然后合并喂给 prompt。每个 store 内部只有向量检索一条路，没有稀疏/关键词这第二条路。

当前代码库中没有 RRF 也没有 BM25 的实现。真正的混合检索应该在单个 store 内部加 BM25 或全文索引。例如 `search_reference_sql` 内部：

```
query = "活跃用户上月总数"
  → dense: embedding → LanceDB vector search
  → sparse: BM25 对 question/sql text 字段做关键词匹配
  → RRF 融合 → 返回
```
