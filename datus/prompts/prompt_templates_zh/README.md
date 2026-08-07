# Datus-Agent 中文提示词模板

本目录包含 56 个已翻译为中文的 Jinja2 (`.j2`) 提示词模板文件，与 `datus/prompts/prompt_templates/` 中的英文原版一一对应。

> **翻译原则**: 所有 Jinja2 模板语法 (`{{ }}`, `{% %}`, `{# #}`)、JSON 键名、SQL 关键字、工具名称和代码标识符均保持原样，仅翻译自然语言指令和说明文字。

---

## 目录

- [一、文件分类总览](#一文件分类总览)
- [二、各类别详细说明](#二各类别详细说明)
  - [1. 共享模板（`_` 前缀）](#1-共享模板-前缀)
  - [2. 核心 SQL 生成](#2-核心-sql-生成)
  - [3. 对话与交互](#3-对话与交互)
  - [4. Schema 与数据发现](#4-schema-与数据发现)
  - [5. 指标与语义模型](#5-指标与语义模型)
  - [6. 可视化、报表与仪表板](#6-可视化报表与仪表板)
  - [7. 评估与质量检查](#7-评估与质量检查)
  - [8. 工作流与运行时上下文](#8-工作流与运行时上下文)
  - [9. 任务专用模板](#9-任务专用模板)
  - [10. 日期解析](#10-日期解析)
- [三、使用方式](#三使用方式)
- [四、版本管理](#四版本管理)
- [五、自定义与扩展](#五自定义与扩展)

---

## 一、文件分类总览

| 类别 | 文件数 | 前缀/标识 | 用途 |
|---|---|---|---|
| 共享模板 | 9 | `_` 开头 | 被其他模板通过 `{% include %}` 引用 |
| 核心 SQL 生成 | 9 | `gen_sql_*`, `fix_sql_*`, `reasoning_*` | NL → SQL 生成、修复、推理 |
| 对话与交互 | 3 | `chat_system_*` | 通用对话、反馈分析 |
| Schema 与数据发现 | 5 | `schema_lineage_*`, `explore_*`, `selection_*` | Schema 链接、数据探索、结果选择 |
| 指标与语义模型 | 6 | `gen_metrics_*`, `gen_semantic_model_*` | 指标定义、语义模型生成 |
| 可视化、报表与仪表板 | 8 | `gen_dashboard_*`, `gen_report_*`, `gen_visual_*`, `gen_table_*`, `visualization_*` | 仪表板、报表、可视化生成 |
| 评估与质量检查 | 6 | `evaluation_*`, `compare_sql_*`, `output_checking_*` | SQL 结果评估、对比、输出检查 |
| 工作流与运行时 | 7 | `plan_mode_*`, `compact_*`, `*_runtime_*`, `memory_*`, `response_*`, `ref_tpl_*` | Plan Mode、压缩、运行时上下文注入 |
| 任务专用 | 7 | `etl_*`, `gen_job_*`, `scheduler_*`, `skill_creator_*`, `ask_*` | ETL、定时任务、技能创建、问答 |
| 日期解析 | 2 | `date_parser_*` | 中英文日期表达式解析 |

---

## 二、各类别详细说明

### 1. 共享模板（`_` 前缀）

这些文件不独立使用，而是通过 `{% include '_xxx.j2' %}` 被其他模板引用，提供跨模板共享的公共内容块。

| 文件 | 行数 | 说明 |
|---|---|---|
| `_ask_artifact_common.j2` | 187 | `ask_dashboard` 和 `ask_report` 系统提示词的共享正文，包含物（artifact）问答的通用指令 |
| `_visual_artifact_common.j2` | 569 | `gen_visual_dashboard` 和 `gen_visual_report` 共享的渲染/数据层/视觉设计指令，是最大的单独模板 |
| `_visual_artifact_context.j2` | 48 | 可视化物的共享上下文块（数据源、schema 等运行时信息） |
| `_osi_dialect_map.j2` | 19 | OSI（Open Semantic Interchange）编译器的 SQL 方言映射宏 |
| `_semantic_sql_history_profiler_gate.j2` | 25 | 可选的 SQL 历史与分布分析说明，用于语义 SQL 历史分析技能 |
| `datasource_runtime_context_1.0.j2` | 8 | 数据源级运行时上下文（仅注入到启用数据库工具的节点） |
| `runtime_context_1.0.j2` | 41 | 通用运行时上下文块，由 `_finalize_system_prompt` 统一追加到所有系统提示词末尾 |
| `response_language_1.0.j2` | 9 | 响应语言设置（根据用户语言自动切换中英文回复） |
| `memory_context_1.0.j2` | 8 | 记忆上下文块（启用记忆功能时注入系统提示词） |

---

### 2. 核心 SQL 生成

这是项目最核心的提示词组，负责 NL → SQL 的整个生命周期。

| 文件 | 行数 | 对应的 Node | 说明 |
|---|---|---|---|
| `gen_sql_system_1.2.j2` | 154 | `GenSQLAgenticNode` | **主 SQL 生成系统提示词**。包含来源优先级、工具使用指南（参考模板、上下文搜索、MetricFlow）、SQL 编写规范、子任务委托等完整指令 |
| `gen_sql_system_1.1.j2` | 10 | `GenSQLAgenticNode` | 旧版兼容别名，指向 `gen_sql_system_1.2.j2` |
| `gen_sql_user_1.0.j2` | 10 | `GenSQLAgenticNode` | SQL 生成任务的用户提示词模板，包含数据库类型、schema、指标、外部知识等上下文 |
| `fix_sql_system_1.0.j2` | 27 | `FixSQLAgenticNode` | SQL 修复系统提示词，分析错误信息并修复 SQL |
| `fix_sql_user_1.0.j2` | 20 | `FixSQLAgenticNode` | SQL 修复的用户提示词，包含错误信息、原始 SQL 和执行结果 |
| `reasoning_system_1.0.j2` | 54 | `ReasonSQLAgenticNode` | SQL 推理系统提示词，迭代生成、执行、优化 SQL |
| `reasoning_user_1.0.j2` | 11 | `ReasonSQLAgenticNode` | SQL 推理的用户提示词 |
| `gen_sql_summary_system_1.1.j2` | 165 | `SQLSummaryAgenticNode` | SQL 摘要与分析提示词，用于知识提取和复用 |
| `ref_tpl_system_1.0.j2` | 49 | 参考模板执行器 | 只能使用预审批的参考模板回答数据问题，无编写 SQL 的能力 |

---

### 3. 对话与交互

通用对话和用户反馈处理的提示词。

| 文件 | 行数 | 对应的 Node | 说明 |
|---|---|---|---|
| `chat_system_1.2.j2` | 72 | `ChatAgenticNode` | **最新版对话系统提示词**。通用 AI 助手，集成数据库/文件系统/上下文搜索/日期解析等工具 |
| `chat_system_1.1.j2` | 58 | `ChatAgenticNode` | 对话系统提示词 1.1 版 |
| `chat_system_0.9.j2` | 150 | `ChatAgenticNode` | 对话系统提示词 0.9 版（最详细版本，包含大量工具使用示例） |
| `feedback_system_1.0.j2` | 39 | `FeedbackAgenticNode` | 对话反馈分析提示词，分析用户满意度并提取改进建议 |

---

### 4. Schema 与数据发现

负责从数据库中检索相关表和列信息。

| 文件 | 行数 | 对应的 Node | 说明 |
|---|---|---|---|
| `schema_lineage_system_1.0.j2` | 42 | `SchemaLinkingNode` | **Schema 链接系统提示词**。分析用户问题和数据库 schema，识别相关表和列 |
| `schema_lineage_user_1.0.j2` | 2 | `SchemaLinkingNode` | Schema 链接的用户提示词，仅包含用户查询 |
| `schema_lineage_summary_1.0.j2` | 39 | `SchemaLinkingNode` | **最终表排序和排除分析**。对候选表进行排名，排除不相关的表，输出最优 schema 子集 |
| `explore_system_1.0.j2` | 93 | `ExploreAgenticNode` | 数据探索提示词。从数据库、知识库和工作空间文件中搜集上下文信息 |
| `selection_analysis_1.0.j2` | 20 | `SelectionNode` | 候选结果分析提示词（并行节点场景），从多个候选中选择最佳结果 |

---

### 5. 指标与语义模型

负责从 SQL 或自然语言中提取业务指标定义和语义模型。

| 文件 | 行数 | 对应的 Node | 说明 |
|---|---|---|---|
| `gen_metrics_system_2.0.j2` | 94 | `GenMetricsAgenticNode` | **最新版指标生成系统提示词**。支持 OSI 和 MetricFlow 双模式，从 SQL 或自然语言创建指标定义 |
| `gen_metrics_system_1.2.j2` | 395 | `GenMetricsAgenticNode` | 指标定义专家 1.2 版。详细的指标创建、更新、管理流程 |
| `gen_metrics_system_1.1.j2` | 275 | `GenMetricsAgenticNode` | 指标定义专家 1.1 版。从多条 SQL 中提取核心指标 |
| `gen_semantic_model_system_2.0.j2` | 106 | `GenSemanticModelAgenticNode` | **最新版语义模型生成系统提示词**。支持 OSI（输出核心 schema 文档）和 MetricFlow（输出 YAML）双模式 |
| `gen_semantic_model_system_1.1.j2` | 416 | `GenSemanticModelAgenticNode` | 语义模型生成 1.1 版。详细的表语义模型创建流程 |
| `gen_table_system_1.0.j2` | 35 | `GenTableAgenticNode` | 数据库表生成提示词。创建表结构、ETL 和数据验证 |

---

### 6. 可视化、报表与仪表板

负责生成数据可视化物（HTML/JSX 格式）。

| 文件 | 行数 | 对应的 Node | 说明 |
|---|---|---|---|
| `gen_dashboard_system_1.0.j2` | 85 | `GenDashboardAgenticNode` | BI 仪表板生成提示词。连接 Superset/Grafana，创建仪表板 |
| `gen_report_system_1.0.j2` | 74 | `GenReportAgenticNode` | 数据分析报告生成提示词 |
| `gen_visual_dashboard_system_1.0.j2` | 377 | `GenVisualDashboardAgenticNode` | **可视化仪表板生成提示词**。输出 React-JSX 仪表板物（header + 筛选器 + KPI 条 + 图表面板） |
| `gen_visual_report_system_1.0.j2` | 252 | `GenVisualReportAgenticNode` | **可视化报表生成提示词**。输出 React-JSX 报表物（长文本文档 + 图表 + 表格） |
| `visualization_system_1.0.j2` | 33 | `VisualizationTool` | 数据可视化推荐提示词，分析 schema 和预览数据并推荐图表类型 |
| `visualization_with_context_1.0.j2` | 30 | `VisualizationTool` | 数据可视化与分析的上下文增强版 |

**相关共享模板**: `_visual_artifact_common.j2`, `_visual_artifact_context.j2`

---

### 7. 评估与质量检查

负责评估 SQL 执行结果的质量。

| 文件 | 行数 | 对应的 Node/Tool | 说明 |
|---|---|---|---|
| `evaluation_2.1.j2` | 30 | `ReflectNode` | **最新版 SQL 评估提示词**。五种分类：SUCCESS / DOC_SEARCH / SIMPLE_REGENERATE / SCHEMA_LINKING / REASONING |
| `evaluation_2.0.j2` | 22 | `ReflectNode` | SQL 评估 2.0 版 |
| `evaluation_1.0.j2` | 18 | `ReflectNode` | SQL 评估 1.0 版 |
| `output_checking_1.0.j2` | 21 | `OutputTool` | **输出检查提示词**。LLM-as-Judge 检查生成的 SQL 是否有逻辑错误、是否需要修正 |
| `compare_sql_system_mcp_1.0.j2` | 34 | `CompareAgenticNode` | SQL 对比系统提示词，分析两个 SQL 查询的差异 |
| `compare_sql_user_1.0.j2` | 31 | `CompareAgenticNode` | SQL 对比用户提示词，包含当前 SQL 和预期结果 |

---

### 8. 工作流与运行时上下文

支持 Plan Mode、对话压缩、运行时上下文注入等功能。

| 文件 | 行数 | 对应的功能 | 说明 |
|---|---|---|---|
| `plan_mode_system_2.0.j2` | 83 | `PlanMode` | **Plan Mode 系统提示词**。基于文件的计划工作流，LLM 用 `plan_tool` 写入 Markdown 计划文件，用户确认后执行 |
| `compact_major_1.0.j2` | 14 | `CompactHook` | **对话压缩提示词**。当上下文过长时，将历史对话压缩为摘要，供新会话恢复 |
| `runtime_context_1.0.j2` | 41 | 运行时注入 | 通用运行时上下文（当前日期、数据源、方言等），由 `_finalize_system_prompt` 追加 |
| `datasource_runtime_context_1.0.j2` | 8 | 运行时注入 | 数据源级运行时上下文，仅注入到启用数据库工具的节点 |
| `memory_context_1.0.j2` | 8 | `MemoryFuncTool` | 记忆上下文，启用记忆功能时注入系统提示词 |
| `response_language_1.0.j2` | 9 | 语言切换 | 根据用户语言设置自动切换回复语言 |

---

### 9. 任务专用模板

各类具体业务场景的专用提示词。

| 文件 | 行数 | 对应的 Node | 说明 |
|---|---|---|---|
| `ask_metrics_system_1.0.j2` | 135 | `AskMetricsAgenticNode` | **指标问答提示词**。以指标优先的方式回答用户的数据问题 |
| `ask_dashboard_system_1.0.j2` | 8 | `AskDashboardAgenticNode` | 仪表板问答提示词（继承 `_ask_artifact_common.j2`） |
| `ask_report_system_1.0.j2` | 8 | `AskReportAgenticNode` | 报表问答提示词（继承 `_ask_artifact_common.j2`） |
| `etl_system_1.1.j2` | 88 | `GenTableAgenticNode` | ETL 任务提示词。从源表构建或更新目标表并验证结果 |
| `gen_job_system_1.0.j2` | 114 | `GenJobAgenticNode` | 数据工程任务提示词。支持库内 ETL 和跨库迁移 |
| `scheduler_system_1.0.j2` | 33 | `SchedulerAgenticNode` | Airflow 定时任务调度提示词 |
| `skill_creator_system_1.0.j2` | 30 | `GenSkillAgenticNode` | 技能工程提示词。创建新技能并优化已有技能 |

---

### 10. 日期解析

| 文件 | 行数 | 对应的 Node | 说明 |
|---|---|---|---|
| `date_parser_en_1.0.j2` | 11 | `DateParserNode` | 英文日期表达式解析（"last month", "Q3 2025" 等） |
| `date_parser_zh_1.0.j2` | 11 | `DateParserNode` | 中文日期表达式解析（"上个月"、"去年第三季度" 等） |

---

## 三、使用方式

### 3.1 模板加载机制

Datus-agent 通过 `datus/prompts/prompt_manager.py` 中的 `PromptManager` 加载和渲染 Jinja2 模板。模板查找顺序：

1. **项目级模板目录**: `{project_root}/.datus/prompts/`（优先级最高）
2. **用户级模板目录**: `~/.datus/prompts/`
3. **内置模板目录**: `datus/prompts/prompt_templates/`（本目录的英文版）

中文模板需要放到上述路径之一才能被加载。

### 3.2 方式一：全局替换为中文

将中文模板复制（或软链接）到用户级模板目录：

```bash
# 备份原有英文模板
cp -r ~/.datus/prompts ~/.datus/prompts_en.bak 2>/dev/null

# 复制中文模板到用户目录
mkdir -p ~/.datus/prompts
cp /path/to/datus/prompts/prompt_templates_zh/*.j2 ~/.datus/prompts/
```

这样所有项目默认使用中文提示词。

### 3.3 方式二：单项目切换

将中文模板复制到项目级模板目录：

```bash
# 在项目根目录下
mkdir -p .datus/prompts
cp datus/prompts/prompt_templates_zh/*.j2 .datus/prompts/
```

仅当前项目受影响，其他项目仍使用英文版。

### 3.4 方式三：自定义混合模式

只复制需要的模板到 `.datus/prompts/`，其他模板仍使用英文版。例如只汉化核心 SQL 生成相关：

```bash
mkdir -p .datus/prompts
cp datus/prompts/prompt_templates_zh/gen_sql_system_1.2.j2 .datus/prompts/
cp datus/prompts/prompt_templates_zh/gen_sql_user_1.0.j2 .datus/prompts/
cp datus/prompts/prompt_templates_zh/evaluation_2.1.j2 .datus/prompts/
cp datus/prompts/prompt_templates_zh/runtime_context_1.0.j2 .datus/prompts/
```

### 3.5 方式四：agent.yml 配置 prompt_version

在 `agent.yml` 中为特定节点指定模板版本：

```yaml
agentic_nodes:
  gen_sql:
    system_prompt: gen_sql          # 使用 gen_sql_system 模板
    prompt_version: "1.2"           # 指定使用 1.2 版本

  chat:
    system_prompt: chat             # 使用 chat_system 模板
    prompt_version: "1.2"

  reflect:
    prompt_version: "2.1"           # 使用 evaluation_2.1.j2
```

### 3.6 验证模板是否正确加载

启动 datus 后，使用 `/model` 或执行一次查询，检查日志输出中的系统提示词内容是否为中文。

也可以使用内置的模板测试命令（如果可用）：

```bash
datus prompt render gen_sql_system --version 1.2 \
  --var has_db_tools=true \
  --var has_context_search_tools=true
```

---

## 四、版本管理

### 4.1 命名规范

模板文件名遵循 `<功能>_<角色>_<主版本>.<次版本>.j2` 格式：

- `gen_sql_system_1.2.j2` → gen_sql 功能的系统提示词，版本 1.2
- `evaluation_2.1.j2` → 评估提示词，版本 2.1
- `chat_system_0.9.j2` → 对话系统提示词，版本 0.9

### 4.2 多版本共存

同一功能可以保留多个版本（如 `evaluation_1.0`、`evaluation_2.0`、`evaluation_2.1`），通过 `agent.yml` 中的 `prompt_version` 字段切换：

```yaml
# 使用最新版
agentic_nodes:
  reflect:
    prompt_version: "2.1"

# 回退到旧版
agentic_nodes:
  reflect:
    prompt_version: "1.0"
```

### 4.3 中文模板与英文原版同步

当英文原版 `datus/prompts/prompt_templates/` 更新后，中文模板需要同步更新。建议：

1. 用 `diff` 对比英文新旧版本，确定变更内容
2. 仅翻译变更部分
3. 保持文件名和版本号与英文版一致

---

## 五、自定义与扩展

### 5.1 创建自定义模板

在 `agent.yml` 中配置 `custom_workflows` 可以引用自定义模板：

```yaml
agentic_nodes:
  gen_sql:
    # 使用自定义模板名
    system_prompt: my_custom_gen_sql
```

然后在 `.datus/prompts/` 下创建 `my_custom_gen_sql_system_1.0.j2`（命名规则：`<system_prompt值>_system_<version>.j2`）。

### 5.2 模板变量参考

不同模板需要不同的上下文变量。以 `gen_sql_system_1.2.j2` 为例：

```jinja2
{% set tools = available_tool_names | default([]) %}
{% set can_execute_sql = "execute_sql" in tools %}
{% set can_search_metrics = "search_metrics" in tools %}
{# ... #}

{% if agent_description %}{{ agent_description }}{% endif %}
{% if rules|length > 0 %}{% for rule in rules %}- {{ rule }}{% endfor %}{% endif %}
```

变量由 `prepare_template_context()` 函数注入（参见 `datus/agent/node/gen_sql_agentic_node.py:874`）。

### 5.3 添加新的共享模板

如果需要创建跨模板共享的中文内容：

1. 创建 `_my_shared_common.j2`（以 `_` 开头标记为共享模板）
2. 在其他模板中通过 `{% include '_my_shared_common.j2' %}` 引用
3. 共享模板放在同一目录下即可被自动发现

---

## 附录 A：文件完整清单

| # | 文件名 | 类别 | 对应英文版 |
|---|---|---|---|
| 1 | `_ask_artifact_common.j2` | 共享 | ✓ |
| 2 | `_osi_dialect_map.j2` | 共享 | ✓ |
| 3 | `_semantic_sql_history_profiler_gate.j2` | 共享 | ✓ |
| 4 | `_visual_artifact_common.j2` | 共享 | ✓ |
| 5 | `_visual_artifact_context.j2` | 共享 | ✓ |
| 6 | `ask_dashboard_system_1.0.j2` | 任务专用 | ✓ |
| 7 | `ask_metrics_system_1.0.j2` | 任务专用 | ✓ |
| 8 | `ask_report_system_1.0.j2` | 任务专用 | ✓ |
| 9 | `chat_system_0.9.j2` | 对话 | ✓ |
| 10 | `chat_system_1.1.j2` | 对话 | ✓ |
| 11 | `chat_system_1.2.j2` | 对话 | ✓ |
| 12 | `compact_major_1.0.j2` | 工作流 | ✓ |
| 13 | `compare_sql_system_mcp_1.0.j2` | 评估 | ✓ |
| 14 | `compare_sql_user_1.0.j2` | 评估 | ✓ |
| 15 | `datasource_runtime_context_1.0.j2` | 共享 | ✓ |
| 16 | `date_parser_en_1.0.j2` | 日期解析 | ✓ |
| 17 | `date_parser_zh_1.0.j2` | 日期解析 | ✓ |
| 18 | `etl_system_1.1.j2` | 任务专用 | ✓ |
| 19 | `evaluation_1.0.j2` | 评估 | ✓ |
| 20 | `evaluation_2.0.j2` | 评估 | ✓ |
| 21 | `evaluation_2.1.j2` | 评估 | ✓ |
| 22 | `explore_system_1.0.j2` | Schema | ✓ |
| 23 | `feedback_system_1.0.j2` | 对话 | ✓ |
| 24 | `fix_sql_system_1.0.j2` | SQL | ✓ |
| 25 | `fix_sql_user_1.0.j2` | SQL | ✓ |
| 26 | `gen_dashboard_system_1.0.j2` | 可视化 | ✓ |
| 27 | `gen_job_system_1.0.j2` | 任务专用 | ✓ |
| 28 | `gen_metrics_system_1.1.j2` | 指标 | ✓ |
| 29 | `gen_metrics_system_1.2.j2` | 指标 | ✓ |
| 30 | `gen_metrics_system_2.0.j2` | 指标 | ✓ |
| 31 | `gen_report_system_1.0.j2` | 可视化 | ✓ |
| 32 | `gen_semantic_model_system_1.1.j2` | 指标 | ✓ |
| 33 | `gen_semantic_model_system_2.0.j2` | 指标 | ✓ |
| 34 | `gen_sql_summary_system_1.1.j2` | SQL | ✓ |
| 35 | `gen_sql_system_1.1.j2` | SQL | ✓ |
| 36 | `gen_sql_system_1.2.j2` | SQL | ✓ |
| 37 | `gen_sql_user_1.0.j2` | SQL | ✓ |
| 38 | `gen_table_system_1.0.j2` | 可视化 | ✓ |
| 39 | `gen_visual_dashboard_system_1.0.j2` | 可视化 | ✓ |
| 40 | `gen_visual_report_system_1.0.j2` | 可视化 | ✓ |
| 41 | `memory_context_1.0.j2` | 共享 | ✓ |
| 42 | `output_checking_1.0.j2` | 评估 | ✓ |
| 43 | `plan_mode_system_2.0.j2` | 工作流 | ✓ |
| 44 | `reasoning_system_1.0.j2` | SQL | ✓ |
| 45 | `reasoning_user_1.0.j2` | SQL | ✓ |
| 46 | `ref_tpl_system_1.0.j2` | SQL | ✓ |
| 47 | `response_language_1.0.j2` | 共享 | ✓ |
| 48 | `runtime_context_1.0.j2` | 共享 | ✓ |
| 49 | `scheduler_system_1.0.j2` | 任务专用 | ✓ |
| 50 | `schema_lineage_summary_1.0.j2` | Schema | ✓ |
| 51 | `schema_lineage_system_1.0.j2` | Schema | ✓ |
| 52 | `schema_lineage_user_1.0.j2` | Schema | ✓ |
| 53 | `selection_analysis_1.0.j2` | Schema | ✓ |
| 54 | `skill_creator_system_1.0.j2` | 任务专用 | ✓ |
| 55 | `visualization_system_1.0.j2` | 可视化 | ✓ |
| 56 | `visualization_with_context_1.0.j2` | 可视化 | ✓ |

---

## 附录 B：模板依赖关系

```
gen_visual_dashboard_system_1.0.j2
    ├── {% include '_visual_artifact_common.j2' %}
    └── {% include '_visual_artifact_context.j2' %}

gen_visual_report_system_1.0.j2
    ├── {% include '_visual_artifact_common.j2' %}
    └── {% include '_visual_artifact_context.j2' %}

ask_dashboard_system_1.0.j2
    └── {% include '_ask_artifact_common.j2' %}

ask_report_system_1.0.j2
    └── {% include '_ask_artifact_common.j2' %}

gen_sql_system_1.1.j2
    └── {% include 'gen_sql_system_1.2.j2' %}  (兼容别名)

gen_semantic_model_system_2.0.j2
    └── {% import '_osi_dialect_map.j2' as dialect_map %}

所有系统提示词
    └── runtime_context_1.0.j2  (由 _finalize_system_prompt 追加)

启用数据库工具的节点
    └── datasource_runtime_context_1.0.j2
```
