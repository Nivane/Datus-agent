# Datus-Agent 源码走读笔记

> 完整对话记录：从项目全景 → `Agent` 入口 → `WorkflowRunner` 调度 → `AgenticNode` 引擎 → `plan.py` 工作流生成 → `evaluate.py` 数据管道 → OpenAI Agents SDK + LiteLLM 双 SDK 架构 → ReAct Agent Loop

---

## 目录

- [一、项目结构全景扫描](#一项目结构全景扫描)
- [二、Agent — 后勤指挥官 (agent.py)](#二agent--后勤指挥官-agentpy)
- [三、WorkflowRunner — 调度员 (workflow_runner.py)](#三workflowrunner--调度员-workflow_runnerpy)
- [四、AgenticNode — 一线战斗员 (agentic_node.py)](#四agenticnode--一线战斗员-agentic_nodepy)
- [五、完整执行路径串联](#五完整执行路径串联)
- [六、Plan — 工作流生成 (plan.py)](#六plan--工作流生成-planpy)
- [七、Reflect — evaluate.py 与双层反思](#七reflect--evaluatepy-与双层反思)
- [八、OpenAI Agents SDK + LiteLLM 双 SDK 架构](#八openai-agents-sdk--litellm-双-sdk-架构)
- [九、ReAct Agent Loop 实现](#九react-agent-loop-实现)
- [十、完整数据流全景](#十完整数据流全景)

---

## 一、项目结构全景扫描

> 详见 [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md)，以下为关键摘要。

### 根目录

| 目录/文件 | 类型 | 用途 |
|---|---|---|
| `datus/` | **核心** | 主源码包，约 500+ 文件 |
| `tests/` | 测试 | unit_tests 1:1 镜像源码，integration/regression 端到端 |
| `ci/` | 构建 | PR/Nightly 测试编排，覆盖率审查 |
| `.github/` | 配置 | 15 个 workflows + Issue/PR 模板 |
| `docs/` | 文档 | MkDocs 中英双语，覆盖 20+ 主题 |
| `conf/` | 配置 | `agent.yml.example`、`providers.yml` |
| `benchmark/` | 测试 | BIRD/Spider2/Semantic Layer 评估 |
| `build_scripts/` | 构建 | PyPI 打包、Docker、测试数据构建 |

### `datus/` 核心模块

| 模块 | 类型 | 用途 |
|---|---|---|
| `datus/agent/` | **核心** | Agent 主类 + Workflow + WorkflowRunner + 49 个 Node |
| `datus/api/` | **核心** | FastAPI 后端：routes/services/models/auth/hooks |
| `datus/cli/` | **核心** | CLI + Textual TUI (84 个文件) |
| `datus/models/` | **核心** | 多 LLM 适配层 (Claude/OpenAI/DeepSeek/Qwen/...) |
| `datus/tools/` | **核心** | 工具层：39 func_tool + db/mcp/skill/permission |
| `datus/storage/` | **核心** | 知识库：LanceDB + SQLite，14 种存储 |
| `datus/prompts/` | **核心** | 提示词管理：60+ Jinja2 模板 |
| `datus/schemas/` | **核心** | Pydantic 数据模式 (43 文件) |
| `datus/configuration/` | **核心** | `agent_config.py` (153KB) 主配置 |
| `datus/gateway/` | **核心** | 飞书/Slack IM 网关 |
| `datus/validation/` | **核心** | 输出校验：hooks + LLM-as-Judge |
| `datus/observability/` | 辅助 | OTLP/Langfuse 追踪 |
| `datus/plugins/` | 辅助 | 插件系统 |
| `datus/auth/` | 辅助 | OAuth2/PKCE 认证 |

### 三大核心文件

| 文件 | 大小 |
|---|---|
| `agentic_node.py` | 197KB |
| `agent_config.py` | 153KB |
| `repl.py` | 122KB |

### ASCII 架构图

```
┌──────────────────────────────────────────────────────────────────┐
│                    ENTRY POINTS                                   │
│  datus-cli  │  datus-api  │  datus-mcp  │  datus-gateway         │
└──────────┬──────────┬──────────┬─────────────────────────────────┘
           ▼          ▼          ▼
┌──────────────────────────────────────────────────────────────────┐
│                 PRESENTATION LAYER                                 │
│  CLI (REPL+TUI)  │  API (routes+services)  │  Gateway (IM)       │
└──────────────────────────┬───────────────────────────────────────┘
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                 ORCHESTRATION LAYER                                │
│  Agent (plan→execute→reflect) → WorkflowRunner → Workflow DAG    │
└──────────────────────────┬───────────────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
     ┌──────────┐  ┌──────────────┐  ┌──────────────┐
     │ PROMPTS  │  │    TOOLS     │  │   STORAGE    │
     │ (Jinja2) │  │  (func_tool) │  │  (LanceDB +  │
     │ 60+ tmpl │  │  39 tools    │  │   SQLite)    │
     └──────────┘  └──────────────┘  └──────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                 INFRASTRUCTURE LAYER                               │
│  Models (Claude/OpenAI/DeepSeek/...)  │  Config (153KB schema)    │
│  Utils (38 modules)  │  Validation (hooks + LLM judge)            │
│  Observability (OTLP/Langfuse)  │  Plugins  │  Auth (OAuth2)     │
└──────────────────────────────────────────────────────────────────┘
                           │
┌──────────────────────────────────────────────────────────────────┐
│                 EXTERNAL INTERFACES                                │
│  SQLite │ DuckDB │ PostgreSQL │ MySQL/ClickHouse/StarRocks/Trino  │
│  MCP Clients │ LLM APIs │ Web Search                              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 二、Agent — 后勤指挥官 (agent.py)

**文件**: `datus/agent/agent.py` (1256 行)

**职责**: 系统主入口。它不直接执行工作流逻辑，而是承担初始化、知识库引导、基准测试编排的"指挥官"角色。

### 2.1 导入与依赖

```
核心依赖链路：
  WorkflowRunner     ← 真正执行工作流的 Runner
  AgentConfig        ← 153KB 的主配置类
  LLMBaseModel       ← LLM 工厂（Claude/OpenAI/...）
  DBManager          ← 多数据库连接管理
  ActionHistory      ← 流式输出的动作记录
```

导入可分成四组：

| 组别 | 导入内容 | 说明 |
|---|---|---|
| **编排层** | `WorkflowRunner` | 负责加载工作流 YAML → 创建节点 → 执行 |
| **配置层** | `AgentConfig`, `BenchmarkConfig` | 全局配置 + 基准测试配置 |
| **模型层** | `LLMBaseModel` | 所有 LLM 的工厂入口 |
| **存储层** | `MetricRAG`, `SemanticModelRAG`, `SchemaWithValueRAG`, `TableSemanticProfileRAG`, `ReferenceSqlRAG`, `ReferenceTemplateRAG` | 6 种知识库 RAG 存储，每个对应 `bootstrap_kb` 中的一个 `component` |

### 2.2 模块级辅助函数

**`_task_item_value()`** — 从基准测试任务字典中安全取字段，`key` 为 `None` 返回 `""`。

**`_connector_context()`** — 尝试从连接器对象上读取 `catalog_name`、`database_name`、`schema_name`，优先调用 `get_current_context()` 方法。返回标准化三要素字典，供 SQL 生成时注入上下文。

**`_resolve_benchmark_sql_context()`** — 合并优先级：**任务级配置 > 连接器当前上下文**。这样 BIRD 等多数据库基准测试中，每个 task 可以指向不同的 database/schema。

### 2.3 `Agent.__init__` — 初始化

```python
def __init__(self, args, agent_config, db_manager=None):
    self.args = args
    self.global_config = agent_config
    if db_manager:
        self.db_manager = db_manager
    else:
        self.db_manager = db_manager_instance(self.global_config.datasource_configs)
    self.tools = {}
    self.storage_modules = {}
    self.metadata_store = None
    self.metrics_store = None
    self._ref_sql_file_sql_counter: Dict[str, int] = {}
    self._print_lock = threading.Lock()
    self._check_storage_modules()
```

| 属性 | 类型 | 用途 |
|---|---|---|
| `self.args` | `argparse.Namespace` | 命令行参数全集 |
| `self.global_config` | `AgentConfig` (153KB) | 全局配置引用 |
| `self.db_manager` | `DBManager` | 传入或用 `db_manager_instance()` 创建（全局单例） |
| `self.tools` | `dict` | 预留的工具字典（目前为空） |
| `self.storage_modules` | `dict` | 标记哪些存储模块已初始化 |
| `self._print_lock` | `threading.Lock` | 多线程打印锁 |

### 2.4 `create_workflow_runner` — Runner 工厂

```python
def create_workflow_runner(self, check_db=True, run_id=None) -> WorkflowRunner:
    return WorkflowRunner(
        self.args, self.global_config,
        pre_run_callable=self.check_db if check_db else None,
        run_id=run_id
    )
```

`pre_run_callable=self.check_db` 是关键——Runner 在执行前会先调用 `Agent.check_db()` 验证数据库连通性。

### 2.5 `run` / `run_stream` — 两种执行模式

```python
# 同步执行
def run(self, sql_task=None, check_storage=False, check_db=True, run_id=None):
    runner = self.create_workflow_runner(check_db=check_db, run_id=run_id)
    return runner.run(sql_task=sql_task, check_storage=check_storage)

# 异步流式执行
async def run_stream(self, sql_task=None, check_storage=False,
                     action_history_manager=None, run_id=None):
    runner = self.create_workflow_runner(run_id=run_id)
    async for action in runner.run_stream(
        sql_task=sql_task, check_storage=check_storage,
        action_history_manager=action_history_manager,
    ):
        yield action
```

两者都委托给 `WorkflowRunner`：
- `run()` → `runner.run()`（同步）
- `run_stream()` → `runner.run_stream()`（异步生成器，CLI REPL 使用）

### 2.6 `check_db` / `probe_llm` — 连通性检查

```python
def check_db(self):
    datasource = self.global_config.current_datasource
    if datasource in self.global_config.datasource_configs:
        connections = self.db_manager.get_connections(datasource)
        if isinstance(connections, dict):
            for name, conn in connections.items():
                conn.test_connection()
        else:
            connections.test_connection()
        return {"status": "success", "message": "Database connection test successful"}

def probe_llm(self):
    llm_model = LLMBaseModel.create_model(model_name="default", agent_config=self.global_config)
    response = llm_model.generate("Hello, can you hear me?")
    return {"status": "success", "message": "LLM model test successful", "response": response}
```

### 2.7 `bootstrap_kb` — 知识库初始化（核心重头戏，~360 行）

根据 `self.args.components` 参数初始化不同的知识库组件：

```python
for component in selected_components:  # ["metadata", "semantic_model", "metrics",
                                        #  "reference_sql", "reference_template"]
    if component == "metadata":
        # 策略：check / overwrite / incremental
        # 分支：本地 / spider2 / bird_dev / bird_critic
        if not benchmark_platform:
            init_local_schema(...)       # 本地数据库
        elif benchmark_platform == "spider2":
            init_snowflake_schema(...)   # Snowflake 格式
        elif benchmark_platform == "bird_dev":
            init_dev_schema(...)         # BIRD 数据集

    elif component == "semantic_model":
        # 三选一：adapter / yaml / success_story
        if uses_adapter:
            init_from_adapter(...)                          # MetricFlow/dbt 等适配器
        elif uses_semantic_yaml:
            init_semantic_yaml_semantic_model(...)          # YAML 文件
        else:
            init_success_story_semantic_model(...)          # 成功案例 CSV

    elif component == "metrics":
        # 三选一同上
        if uses_adapter:
            init_from_adapter(...)
        elif uses_semantic_yaml:
            init_semantic_yaml_metrics(...)
        else:
            init_success_story_metrics(..., emit=self._emit_metrics_event)

    elif component == "reference_sql":
        self._reset_reference_sql_stream_state()
        result = init_reference_sql(..., emit=self._emit_reference_sql_event)

    elif component == "reference_template":
        self._reset_reference_template_stream_state()
        result = init_reference_template(..., emit=self._emit_reference_template_event)
```

### 2.8 `do_benchmark` — 基准测试执行器

```python
def do_benchmark(self, benchmark_platform, target_task_ids=None, run_id=None):
    self.check_db()

    def run_single_task(task_id, benchmark_config, task_item):
        task_datasource = _task_item_value(task_item, benchmark_config.datasource_key)
        task_conn = db_manager.get_conn(task_datasource, task_database)
        sql_context = _resolve_benchmark_sql_context(benchmark_config, task_item, task_conn)

        result = self.run(
            SqlTask(id=task_id, datasource=task_datasource,
                    database_type=task_conn.dialect, task=task,
                    catalog_name=sql_context["catalog_name"],
                    database_name=sql_context["database_name"],
                    schema_name=sql_context["schema_name"],
                    ...),
            check_storage=False, check_db=False, run_id=run_id,
        )
        return task_id, result

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        for task_item in load_benchmark_tasks(...):
            f = executor.submit(run_single_task, task_id, benchmark_config, task_item)
    # Wait for completion
    for future in as_completed(future_to_task): ...
```

### 2.9 架构定位

```
agent.py 在系统中的位置:

CLI / API / Gateway (用户交互入口)
        │ 调用
        ▼
Agent (agent.py)
  职责:
  • bootstrap_kb()  知识库初始化
  • check_db()      连通性检查
  • probe_llm()     LLM 测试
  • benchmark()     基准测试编排
  • evaluation()    结果评估
  • generate_dataset() 训练数据导出

  不负责:
  ✗ 实际工作流执行 → 委托给 WorkflowRunner
  ✗ LLM 调用细节    → 委托给 AgenticNode
  ✗ SQL 生成/执行   → 委托给对应 Node
        │ 委托
        ▼
WorkflowRunner (加载 workflow.yml → 创建节点 → 执行)
```

**一句话**: Agent 是"后勤指挥官"——它在开战前把武器库（知识库）、通信线路（数据库）、情报系统（LLM）都准备好，然后派 WorkflowRunner 上前线执行具体任务。

---

## 三、WorkflowRunner — 调度员 (workflow_runner.py)

**文件**: `datus/agent/workflow_runner.py` (380 行)

**职责**: Agent 和 Workflow 之间的桥梁。负责 DAG 调度循环。

### 3.1 核心依赖

```
导入链：
  generate_workflow  ← plan.py: 根据 task 生成 Workflow DAG
  Workflow           ← workflow.py: 工作流数据结构 + 节点遍历
  evaluate_result    ← evaluate.py: 每步执行后评估结果质量
  setup_node_input   ← evaluate.py: 为下一个节点准备输入数据
  ActionHistory      ← 流式输出的动作记录
  SqlTask            ← 用户输入的任务数据结构
```

### 3.2 `__init__` — 轻量初始化

```python
class WorkflowRunner:
    def __init__(self, args, agent_config, *, pre_run_callable=None, run_id=None):
        self.workflow: Optional[Workflow] = None   # 核心：工作流实例
        self.workflow_ready = False                 # 状态标记
        self._pre_run = pre_run_callable            # 执行前钩子（Agent.check_db）
        self.run_id = run_id
```

**关键设计**: `WorkflowRunner` 是无状态的壳——不持有什么重资源。可以每次执行创建一个新 Runner。

### 3.3 工作流生命周期 — 三条路径

**路径 1: 新建 → `initialize_workflow(sql_task)`**

```python
def initialize_workflow(self, sql_task: SqlTask):
    plan_type = getattr(self.args, "workflow", None) or self.global_config.workflow_plan
    self.workflow = generate_workflow(
        task=sql_task, plan_type=plan_type, agent_config=self.global_config,
    )
    if hasattr(self.args, "plan_mode"):
        plan_mode = bool(self.args.plan_mode)
        self.workflow.metadata["plan_mode"] = plan_mode
        self.workflow.metadata["auto_execute_plan"] = plan_mode  # headless 自动批准
    self.workflow.display()
```

**路径 2: 恢复 → `resume_workflow(config)`**

```python
def resume_workflow(self, config):
    self.workflow = Workflow.load(config.load_cp)  # 从 YAML checkpoint 反序列化
    self.workflow.global_config = self.global_config
    self.workflow.resume()  # 跳过已完成的节点，定位到中断点
```

**路径 3: 统一入口 → `init_or_load_workflow(sql_task)`**

```python
if args.load_cp:
    resume_workflow(self.args)       # 从检查点恢复
elif sql_task:
    initialize_workflow(sql_task)    # 新建
elif not self.workflow_ready:
    报错                              # 什么都没给
```

### 3.4 `run()` — 同步执行循环

```python
@optional_traceable(name="agent", context_builder=...)
def run(self, sql_task=None, check_storage=False) -> Dict:
    logger.info("Starting agent execution")
    if not self._ensure_prerequisites(sql_task, check_storage):
        return {}

    step_count = 0
    max_steps = self.args.max_steps or 20
    self._prepare_first_node()

    while self.workflow and not self.workflow.is_complete() and step_count < max_steps:
        current_node = self.workflow.get_current_node()
        if not current_node:
            logger.warning("No more tasks to execute. Exiting.")
            break

        logger.info(f"Executing task: {current_node.description}")
        current_node.run()               # ← 同步入口 → 内部调 execute_stream()

        if current_node.status == "failed":
            if current_node.type == NodeType.TYPE_PARALLEL:
                # 并行节点：检查是否有任何子节点成功
                has_any_success = False
                if current_node.result and hasattr(current_node.result, "child_results"):
                    for v in current_node.result.child_results.values():
                        ok = v.get("success", False) if isinstance(v, dict) else getattr(v, "success", False)
                        if ok:
                            has_any_success = True
                            break
                if has_any_success:
                    logger.warning("Parallel node partial failure, continue to selection")
                else:
                    logger.warning(f"Parallel node all failed: {current_node.description}")
                    break
            else:
                logger.warning(f"Node failed: {current_node.description}")
                break

        evaluation = evaluate_result(current_node, self.workflow)  # ← 数据管道
        logger.debug(f"Evaluation result for {current_node.type}: {evaluation}")
        if not evaluation["success"]:
            logger.error(f"Setting {current_node.type} status to failed due to evaluation failure")
            current_node.status = "failed"
            break

        self.workflow.advance_to_next_node()
        step_count += 1

    if step_count >= max_steps:
        logger.warning(f"Workflow execution stopped after reaching max steps: {max_steps}")

    metadata = self._finalize_workflow(step_count)
    return metadata.get("final_result", {})
```

**关键细节—并行节点容错**：

```python
if current_node.type == NodeType.TYPE_PARALLEL:
    has_any_success = any(child succeeded)
    if has_any_success:
        continue   # 部分失败可以接受
    else:
        break      # 全部失败才中断
```

### 3.5 `run_stream()` — 流式执行循环

与 `run()` 核心逻辑相同，区别：

- 异步生成器，逐个 `yield ActionHistory`
- 三阶段包裹：`workflow_init` → `node_execution_{id}` × N → `workflow_completion`
- 节点内部流式转发：`async for node_action in current_node.run_stream(action_history_manager): yield`
- 每个阶段先 yield `PROCESSING`，再更新为 `SUCCESS/FAILED`（前端进度条用）

```python
@optional_traceable(name="agent_stream", ...)
async def run_stream(self, sql_task=None, check_storage=False,
                     action_history_manager=None) -> AsyncGenerator[ActionHistory, None]:
    # 阶段 1: 初始化
    init_action = ActionHistory(..., action_type="workflow_init", status=PROCESSING)
    yield init_action
    if not self._ensure_prerequisites(sql_task, check_storage):
        init_action.status = FAILED; return
    init_action.status = SUCCESS; yield init_action

    # 阶段 2: 节点执行
    while self.workflow and not self.workflow.is_complete() and step_count < max_steps:
        node_start_action = ActionHistory(..., action_type="node_execution", status=PROCESSING)
        yield node_start_action

        try:
            async for node_action in current_node.run_stream(action_history_manager):
                yield node_action  # 透明转发节点内部事件

            if current_node.status == "failed":
                node_start_action.status = FAILED; break
            node_start_action.status = SUCCESS

        except Exception as e:
            node_start_action.status = FAILED; break

        evaluate_result(...)
        advance_to_next_node()

    # 阶段 3: 收尾
    completion_action = ActionHistory(..., action_type="workflow_completion")
    yield completion_action
    metadata = self._finalize_workflow(step_count)
    completion_action.output = metadata; yield completion_action
```

### 3.6 与 Agent 的委托关系

```python
# Agent.run()
runner = self.create_workflow_runner(check_db=check_db, run_id=run_id)
return runner.run(sql_task=sql_task, check_storage=check_storage)

# Agent.run_stream()
runner = self.create_workflow_runner(run_id=run_id)
async for action in runner.run_stream(sql_task=sql_task, ...):
    yield action
```

**一句话**: WorkflowRunner 是"胶水层"——把 Plan（读配置）、Execute（驱动节点）、Reflect（数据交接）串成流水线。

---

## 四、AgenticNode — 一线战斗员 (agentic_node.py)

**文件**: `datus/agent/node/agentic_node.py` (4082 行，项目最大文件)

**职责**: 所有具体节点的父类。持有 LLM 连接、工具集、会话记忆，驱动真正的 LLM 推理和工具调用。

### 4.1 类图与继承关系

```
Node  (node.py — 基础抽象)
  └── AgenticNode  (本文主角)
        ├── ChatAgenticNode              (对话)
        ├── GenSQLAgenticNode            (NL → SQL)
        ├── GenMetricsAgenticNode        (指标生成)
        ├── GenDashboardAgenticNode      (仪表板生成)
        ├── GenReportAgenticNode         (报表生成)
        ├── GenSemanticModelAgenticNode  (语义模型生成)
        ├── AskMetricsAgenticNode        (指标问答)
        ├── AskDashboardAgenticNode      (仪表板问答)
        ├── AskReportAgenticNode         (报表问答)
        ├── ExploreAgenticNode           (数据探索)
        ├── CompareAgenticNode           (对比分析)
        ├── FeedbackAgenticNode          (反馈收集)
        ├── SQLSummaryAgenticNode        (SQL 摘要)
        ├── SchedulerAgenticNode         (定时调度)
        ├── BaseArtifactAskAgenticNode   (可视化物问答基类)
        ├── BaseVisualArtifactAgenticNode (可视化物生成基类)
        └── DeliverableAgenticNode       (中间抽象)
```

### 4.2 `__init__` — 220 行初始化

**第 1 层: 基础**

```python
super().__init__(node_id, description, node_type, input_data, agent_config, tools)
self.scope = scope          # 用户/租户隔离
self.mcp_servers = {}        # MCP 服务端连接
self.actions = []            # 本节点的动作历史
```

**第 2 层: 会话系统**

```python
self.session_id: str          # 唯一会话 ID（UUID），自生成或传入
self._session: AdvancedSQLiteSession  # OpenAI Agents SDK 的 SQLite 会话
self.session_subdir: str      # 子 agent 的额外路径层
```

会话存储路径布局：

```
{session_dir} / {scope} / {session_subdir} / {session_id}.db
                      ↑ 用户/租户        ↑ 子 agent 时= 父 session_id
```

**第 3 层: 模型管理（懒加载）**

```python
self._agent_config_ref = agent_config
self._node_model_name = ...              # 节点级 model 覆盖
self._pinned_model = None                # 外部注入 mock，优先级最高
```

核心属性 `model` 的解析优先级：

```python
@property
def model(self) -> Optional[LLMBaseModel]:
    if self._pinned_model is not None:      # ① 外部注入（测试用）
        return self._pinned_model
    return LLMBaseModel.create_model(       # ② 走 LRU 缓存工厂
        agent_config=self._agent_config_ref,
        model_name=self._node_model_name,   # ③ 节点级覆盖 or "default"
        scope=self.scope,
    )
```

每次访问都走 `create_model()`，但有进程级 LRU 缓存。

**第 4 层: 工具与权限（全部懒加载）**

```python
self.permission_manager   # 权限管理器
self.permission_hooks     # 权限钩子（拦截工具调用）
self.skill_manager        # 技能生命周期
self.skill_func_tool      # 技能函数工具
self.ask_user_tool         # 向用户提问工具
self.sub_agent_task_tool   # 子 Agent 分发
self.bash_tool             # Bash 执行工具
self.memory_func_tool      # 记忆读写
```

**第 5 层: 基础设施**

```python
self.action_bus            # 动作总线（合并多源事件流）
self.tool_channel          # 代理工具通道（print_mode 用）
self.tool_registry         # 工具名→类别注册
self.interaction_broker    # 外部交互代理（子 agent 请求用户确认）
self.interrupt_controller  # Ctrl+C 中断控制器
self.pending_input_queue   # 交互模式下的用户输入队列
```

**第 6-8 层: Plan Mode + Compact + 收尾**

```python
self.plan_mode_active = False
self.plan_file_path               # 计划文件 .md 路径
self._compact_cfg                 # 压缩配置
self._compacted_until = 0         # 下次 minor compact 扫描起点
self.running_turn_usage           # 当前 turn 的 Token 使用量

# 收尾
if not self.session_id:
    self.session_id = self._generate_session_id()
self.restore_plan_mode_state()    # 恢复计划模式状态
self.restore_context_state()      # 恢复上下文窗占用率
```

### 4.3 `execute()` — 同步包装器

```python
def execute(self) -> BaseResult:
    action_history_manager = ActionHistoryManager()

    async def _run_async():
        final_action = None
        async for action in self.execute_stream(action_history_manager):
            if action.status == ActionStatus.SUCCESS:
                final_action = action
        return final_action

    try:
        final_action = asyncio.run(_run_async())  # 隔离事件循环

        if final_action and final_action.output:
            output_data = final_action.output
            if isinstance(output_data, dict):
                result_class = getattr(self, "result_class", None)
                if result_class:
                    self.result = result_class.model_validate(output_data)  # Pydantic 验证
                else:
                    self.result = BaseResult(success=output_data.get("success", True), ...)
        if not self.result:
            self.result = BaseResult(success=False, error="No result from execution")
        return self.result

    except Exception as e:
        self.result = BaseResult(success=False, error=str(e))
        return self.result
```

### 4.4 `execute_stream()` — 模板方法（final，子类禁止重写）

```python
async def execute_stream(self, action_history_manager=None):
    """
    Template method — subclasses MUST NOT override.
    Customize via: _before_stream, _build_template_context, _compose_run_hooks,
    _get_retry_policy, _build_success_result, _stream_post_build
    """
    ctx = StreamRunContext(
        user_input=self.input,
        action_history_manager=ahm,
        pending_input_queue=getattr(self, "pending_input_queue", None),
    )

    # 1. yield 初始 USER action
    initial_action = ActionHistory.create_action(
        role=ActionRole.USER,
        action_type=f"{node_name}_request",
        messages=f"User: {getattr(self.input, 'user_message', '')}",
        input_data=self.input.model_dump(),
    )
    ahm.add_action(initial_action)
    yield initial_action

    try:
        # 2. 钩子: 流开始前
        await self._before_stream(ctx)

        # 3. 自动压缩 + 会话
        await self._auto_compact()
        ctx.session = self._get_or_create_session()

        # 4. 系统提示词（带版本化缓存）
        ctx.system_instruction = self._get_session_system_prompt(
            prompt_version=getattr(self.input, "prompt_version", None),
            template_context=self._build_template_context(ctx),
        )

        # 5. 用户提示词
        if ctx.user_message_override is not None:
            original = self.input.user_message
            self.input.user_message = ctx.user_message_override
            try:
                ctx.user_prompt = self._build_enhanced_message(self.input)
            finally:
                self.input.user_message = original
        else:
            ctx.user_prompt = self._build_enhanced_message(self.input)

        # 6. 重试循环
        policy = self._get_retry_policy() or NoRetryPolicy()
        max_attempts = max(1, getattr(policy, "max_attempts", 1))

        for attempt in range(1, max_attempts + 1):
            ctx.attempt = attempt
            policy.reset(ctx)

            async for stream_action in self._stream_once(ctx):
                yield stream_action

            wants_retry = policy.should_retry(ctx)
            if not wants_retry or attempt >= max_attempts:
                break
            for retry_action in policy.on_retry_actions(ctx):
                ahm.add_action(retry_action)
                yield retry_action
            next_prompt = policy.next_prompt(ctx)
            if next_prompt is not None:
                ctx.user_prompt = next_prompt

        policy.finalise(ctx)

        # 7. 成功结果
        result = self._build_success_result(ctx)
        self.result = result

        # 8. 可选后处理钩子
        async for progress_action in self._stream_post_build(ctx, result):
            ahm.add_action(progress_action)
            yield progress_action

        # 9. yield 最终 ASSISTANT action
        final_action = ActionHistory.create_action(
            role=ActionRole.ASSISTANT,
            action_type=f"{node_name}_response",
            messages=...,
            input_data=self.input.model_dump(),
            output_data=result.model_dump(),
            status=ActionStatus.SUCCESS if getattr(result, "success", True) else ActionStatus.FAILED,
        )
        ahm.add_action(final_action)
        yield final_action

    except ExecutionInterrupted:
        self._drop_running_turn_usage_on_exit = False  # Ctrl+C 保留部分数据
        raise
    except Exception as exc:
        error_result = self._build_error_result(exc, ctx)
        self.result = error_result
        error_action = ActionHistory.create_action(role=ActionRole.ASSISTANT,
            action_type="error", messages=f"{node_name} interaction failed: {error_msg}",
            output_data=error_result.model_dump(), status=ActionStatus.FAILED,)
        ahm.add_action(error_action)
        yield error_action
    finally:
        # 清理 running_turn_usage
        if getattr(self, "_drop_running_turn_usage_on_exit", True):
            self.running_turn_usage = None
```

生命周期图解：

```
execute_stream() 生命周期:

  1. 创建 StreamRunContext
  2. yield 初始 USER action
  ┌──────────────────────────────────────────┐
  │ 3. await _before_stream(ctx)      ← 钩子 │
  │ 4. await _auto_compact()          ← 压缩 │
  │ 5. ctx.session = _get_or_create_session()│
  │ 6. ctx.system_instruction =              │
  │      _get_session_system_prompt()        │
  │ 7. ctx.user_prompt =                     │
  │      _build_enhanced_message()           │
  │                                          │
  │ 8. for attempt in 1..max_attempts:       │
  │      _stream_once(ctx)     ← LLM+工具    │
  │      policy.should_retry? → 循环/退出    │
  │                                          │
  │ 9. result = _build_success_result(ctx)   │
  │10. _stream_post_build(ctx, result)       │
  │11. yield final ASSISTANT action          │
  └──────────────────────────────────────────┘
```

### 4.5 `_stream_once()` — LLM 单次调用

```python
async def _stream_once(self, ctx: "StreamRunContext") -> AsyncGenerator[ActionHistory, None]:
    explicit_turns = (
        getattr(ctx.user_input, "max_turns", None)
        if "max_turns" in getattr(ctx.user_input, "model_fields_set", ())
        else None
    )
    effective_max_turns = explicit_turns if explicit_turns is not None else self.max_turns

    self._ensure_tool_transformers()
    self._current_action_history = ctx.action_history_manager

    try:
        async for stream_action in self.model.generate_with_tools_stream(
            prompt=ctx.user_prompt,
            tools=self.tools or [],
            mcp_servers=self.mcp_servers,
            instruction=ctx.system_instruction,
            max_turns=effective_max_turns,
            session=ctx.session,
            action_history_manager=ctx.action_history_manager,
            hooks=self._compose_run_hooks(ctx),
            agent_name=self.get_node_name(),
            interrupt_controller=self.interrupt_controller,
            pending_input_queue=ctx.pending_input_queue,
            interaction_broker=getattr(self, "interaction_broker", None),
        ):
            rewritten = self._maybe_rewrite_stream_action(stream_action, ctx)
            action_to_yield = rewritten or stream_action

            # 分类处理:
            # ASSISTANT + SUCCESS → 提取 response_content（过滤 is_thinking）
            if action_to_yield.role == ActionRole.ASSISTANT and action_to_yield.status == ActionStatus.SUCCESS:
                output = action_to_yield.output
                if isinstance(output, dict) and output.get("is_thinking") is not True:
                    ctx.last_successful_output = output
                    candidate = output.get("content", "") or output.get("response", "") or output.get("raw_output", "")
                    if isinstance(candidate, str) and candidate:
                        ctx.response_content = candidate

            # TOOL + SUCCESS → 提取 tool_summary
            elif action_to_yield.role == ActionRole.TOOL and action_to_yield.status == ActionStatus.SUCCESS:
                tool_output = action_to_yield.output if isinstance(action_to_yield.output, dict) else {}
                summary = tool_output.get("summary") or tool_output.get("status_message") or ""
                if isinstance(summary, str) and summary.strip():
                    ctx.last_tool_summary = summary.strip()

            yield action_to_yield
    finally:
        self._current_action_history = None
```

### 4.6 `_compose_hooks()` — 钩子组合器

```python
def _compose_hooks(self, extra: Any = None) -> Any:
    self._ensure_permission_hooks()
    self._ensure_tool_transformers()
    compact_hook = self._get_or_create_compact_hook()
    token_usage_hook = self._get_or_create_token_usage_hook()

    active = [h for h in (extra, self.permission_hooks, compact_hook, token_usage_hook) if h is not None]
    if not active:
        return None
    if len(active) == 1:
        return active[0]
    return CompositeHooks(active)
```

四个钩子的作用：

| 钩子 | 触发时机 | 作用 |
|---|---|---|
| `permission_hooks` | 工具调用前 | 拦截危险 bash / SQL，弹出确认对话框 |
| `compact_hook` | 工具调用后 | 累计计数器，超过阈值触发对话压缩 |
| `token_usage_hook` | LLM 返回后 | 更新状态栏的 Token 用量 |
| `extra` | 各处 | 节点特定钩子（如 `TodoListHook`） |

### 4.7 `execute_stream_with_interactions()` — REPL 专用

```python
async def execute_stream_with_interactions(self, action_history_manager=None):
    self.interrupt_controller.reset()
    self.action_bus.reset()
    broker = self._get_or_create_broker()

    action_stream = self.execute_stream(action_history_manager)
    try:
        # 三流合一: LLM 事件 + 外部交互事件
        async for action in self.action_bus.merge(
            action_stream,           # 主 LLM 事件流
            broker.fetch(),          # 外部交互事件（如子 agent 请求用户确认）
            on_primary_done=broker.close,
        ):
            self.interrupt_controller.check()  # Ctrl+C 检测
            yield action
    except ExecutionInterrupted:
        yield ActionHistory.create_action(
            role=ActionRole.ASSISTANT,
            action_type="interrupted",
            messages="Execution interrupted. You can continue with additional information.",
            status=ActionStatus.SUCCESS,
        )
```

### 4.8 8 个子类钩子点

| 钩子 | 签名 | 作用 | 典型覆盖者 |
|---|---|---|---|
| `_before_stream` | `async (ctx)` | 流开始前设置 | `CompareAgenticNode` |
| `_build_template_context` | `(ctx) → dict` | 向系统提示注入变量 | `GenSQLAgenticNode`：注入 schema |
| `_compose_run_hooks` | `(ctx) → hooks` | 提供节点级钩子 | `GenReportAgenticNode`：`TodoListHook` |
| `_maybe_rewrite_stream_action` | `(action, ctx) → action` | 实时改写动作 | `GenReportAgenticNode`：JSON→Markdown |
| `_get_retry_policy` | `() → policy` | 重试策略 | `GenSQLAgenticNode`：SQL 校验重试 |
| `_build_success_result` | `(ctx) → BaseResult` | **必须覆盖**—构造结果 | 每个子类 |
| `_stream_post_build` | `async (ctx, result)` | 结果后处理 | `BaseVisualArtifactAgenticNode` |
| `DEFAULT_SKILLS` | `str` (类变量) | 默认加载的技能 | `GenDashboardAgenticNode` |

### 4.9 三个核心设计哲学

1. **模板方法模式**: `execute_stream()` 是 `final` 的——子类通过钩子定制，不改主流程。注释写道 `Subclasses MUST NOT override execute_stream itself — the template contract is final.`
2. **懒加载一切**: model, session_manager, permission_hooks, skill_manager, bash_tool 全部懒加载
3. **会话隔离**: 每个 session_id → 独立 SQLite db，子 agent 嵌套在父目录下

**一句话**: AgenticNode 是"一线战斗员"——拿着 LLM 去对话、调工具、生成 SQL / 指标 / 报表 / 仪表板。

---

## 五、完整执行路径串联

回顾从用户输入到返回结果的全链路：

```
用户输入 "show me sales by region"
        │
        ▼
Agent.run(SqlTask)
        │
        ▼
WorkflowRunner.run()
        │
        ├─ generate_workflow()         ← plan.py: 根据 task+plan_type 生成节点 DAG
        │
        ├─ while not complete:         ← 循环调度
        │     node = workflow.get_current_node()
        │     node.run() ─────────┐
        │                         │
        │                         ▼
        │               AgenticNode.execute()
        │                         │
        │                         ▼
        │               AgenticNode.execute_stream()
        │                         │
        │          ┌──────────────┼──────────────┐
        │          ▼              ▼              ▼
        │   system_prompt    user_prompt     session
        │   (Jinja2渲染)   (NL+上下文)    (历史对话)
        │          └──────────────┼──────────────┘
        │                         │
        │                         ▼
        │            model.generate_with_tools_stream()
        │                         │
        │          ┌──────────────┼──────────────┐
        │          ▼              ▼              ▼
        │      LLM推理       工具调用        钩子触发
        │    (生成文本)   (查DB/搜索/Bash) (权限/压缩/Token)
        │                         │
        │                         ▼
        │            ctx.response_content
        │                         │
        │                         ▼
        │          _build_success_result(ctx)
        │                         │
        │     ◄───────────────────┘
        │
        │     evaluate_result()          ← 数据管道: 节点N → workflow.context → 节点N+1
        │     advance_to_next_node()
        │
        ▼
_finalize_workflow() → save trajectory → return final_result
```

---

## 六、Plan — 工作流生成 (plan.py)

**文件**: `datus/agent/plan.py` (278 行)

### 6.1 核心发现: workflow 不是 LLM 动态生成的！

```python
# plan.py:210
def generate_workflow(task: SqlTask, plan_type: str = "reflection",
                      agent_config: Optional[AgentConfig] = None) -> Workflow:
    logger.info(f"Generating workflow for task based on plan type '{plan_type}': {task}")

    if not plan_type and agent_config:
        plan_type = agent_config.workflow_plan
    elif not plan_type:
        plan_type = "reflection"  # fallback

    # ① 优先查 agent.yml 的 custom_workflows
    if agent_config and plan_type in agent_config.custom_workflows:
        selected_workflow = agent_config.custom_workflows[plan_type]
    else:
        # ② 回退到 datus/agent/workflow.yml 内置 workflows
        config = load_builtin_workflow_config()
        workflows = config.get("workflow", {})
        if plan_type not in workflows:
            raise ValueError(f"Invalid plan type '{plan_type}'.")
        selected_workflow = workflows[plan_type]

    # ③ create_nodes_from_config() → 逐条解析 YAML → Node.new_instance()
    # 没有任何 LLM 调用
    nodes = create_nodes_from_config(workflow_steps, task, agent_config, workflow.tools)
    for node in nodes:
        workflow.add_node(node)
    return workflow
```

### 6.2 `workflow.yml` — 所有内置工作流是静态列表

```yaml
workflow:

  reflection:                    # ← 最常用
    - schema_linking
    - gen_sql
    - execute_sql
    - reflect
    - output

  fixed:                         # ← 无反思快速模式
    - schema_linking
    - gen_sql
    - execute_sql
    - output

  empty: []

  dynamic:                       # ← 同 reflection
    - schema_linking
    - gen_sql
    - execute_sql
    - reflect
    - output

  metric_to_sql:                 # ← 指标转 SQL
    - schema_linking
    - search_metrics
    - date_parser
    - gen_sql
    - execute_sql
    - output

  chat_agentic:                  # ← 对话模式
    - chat
    - execute_sql
    - output

  gen_sql_agentic:               # ← 纯 SQL 生成
    - gen_sql
    - execute_sql
    - output
```

### 6.3 `create_nodes_from_config` — 字符串 → 节点实例

```python
def create_nodes_from_config(workflow_config, sql_task, agent_config, tools):
    nodes = []
    # 第一个节点：begin_node（自动完成）
    start_node = Node.new_instance(
        node_id="node_0", node_type=NodeType.TYPE_BEGIN, input_data=sql_task, ...)
    nodes.append(start_node)

    for item in config:
        if isinstance(item, str):       # "gen_sql" → GenSQLAgenticNode
            node = _create_single_node(item, node_id, sql_task, agent_config)
            nodes.append(node)

        elif isinstance(item, dict):    # {"parallel": [...]}  → ParallelNode
            for key, value in item.items():
                if key == "parallel":
                    parallel_node = Node.new_instance(
                        node_type=NodeType.TYPE_PARALLEL,
                        input_data=ParallelInput(child_nodes=value), ...)
                elif key == "selection":
                    selection_node = Node.new_instance(
                        node_type=NodeType.TYPE_SELECTION,
                        input_data=SelectionInput(selection_criteria=...), ...)
    return nodes
```

### 6.4 `_create_single_node` — 别名标准化 + 节点构造

```python
def _create_single_node(node_type, node_id, sql_task, agent_config):
    # 别名标准化
    if node_type in {"reason_sql", "reasoning_sql", "reason"}:
        normalized_type = NodeType.TYPE_REASONING
    elif node_type in {"reflection", "reflect"}:
        normalized_type = NodeType.TYPE_REFLECT
    elif node_type == "execute":
        normalized_type = NodeType.TYPE_EXECUTE_SQL
    elif node_type == "chat":
        normalized_type = NodeType.TYPE_CHAT
    # 查 agentic_nodes 配置
    elif agent_config and node_type in agent_config.agentic_nodes:
        agentic_config = agent_config.agentic_nodes[node_type]
        normalized_type = agentic_config.get("node_type")

    # 特殊节点需要构建专属 Input
    if normalized_type == NodeType.TYPE_SCHEMA_LINKING:
        input_data = SchemaLinkingInput.from_sql_task(sql_task, ...)
    elif normalized_type == NodeType.TYPE_ASK_METRICS:
        input_data = AskMetricsNodeInput(user_message=sql_task.task, ...)

    return Node.new_instance(node_id=node_id, node_type=normalized_type,
                             input_data=input_data, agent_config=agent_config, ...)
```

### 6.5 YAML 字符串 → 节点实例映射

| YAML 字符串 | 最终节点类 |
|---|---|
| `"schema_linking"` | `SchemaLinkingAgenticNode` |
| `"gen_sql"` | `GenSQLAgenticNode` |
| `"execute_sql"` | `ExecuteSQLNode` |
| `"reflect"` / `"reflection"` | `ReflectNode` |
| `"chat"` | `ChatAgenticNode` |
| `"output"` | `OutputNode` |
| `"search_metrics"` | `SearchMetricsNode` |
| `"date_parser"` | `DateParserNode` |
| `"reason_sql"` / `"reasoning_sql"` | `ReasonSQLNode` |

### 6.6 Plan Mode ≠ LLM 生成工作流

Plan mode 是"让 LLM 在执行前先写一份 Markdown 计划文件"，不是让 LLM 生成工作流 DAG：

```
普通模式:
  用户: "帮我查销售额"
  → schema_linking → gen_sql → execute_sql → output
  (LLM 在每个节点里干活，但节点顺序是 YAML 决定的)

Plan Mode (--plan_mode):
  用户: "帮我做销售分析"
  → Turn 1: LLM 用 plan_tool 写 .md 计划文件:
     "1. 了解表结构  2. 查月度趋势  3. 对比去年同期"
  → 用户确认 (confirm_plan)
  → Turn 2: 仍然走 reflection DAG，但计划文件注入系统提示词
```

核心代码：

```python
# agentic_node.py:428-470
def activate_plan_mode(self) -> str:
    self.plan_mode_active = True
    self.workflow_prompt_sent = False
    plan_dir = os.path.join(self._resolve_workspace_root(), ".datus", "plans")
    self.plan_file_path = os.path.join(plan_dir, f"{uuid.uuid4().hex[:8]}.md")
    # 预创建空计划文件 → LLM 可以用 read_file 探测后用 write_file 写入

# agentic_node.py:655-670
def _get_plan_mode_tools(self) -> List[Tool]:
    self.plan_tool = PlanTool(self._session, session_id=lambda: self.session_id)
    self.confirm_plan_tool = ConfirmPlanTool(self)
    tools = list(self.plan_tool.available_tools())
    tools.extend(self.confirm_plan_tool.available_tools())
    return tools
```

### 6.7 自定义工作流

```yaml
# agent.yml
workflow_plan: my_parallel

custom_workflows:
  my_parallel:
    steps:
      - schema_linking
      - parallel:              # 并行执行！
          - gen_sql
          - search_metrics
      - selection              # 选最优结果
      - execute_sql
      - output
    config:
      max_retries: 2
```

---

## 七、Reflect — evaluate.py 与双层反思

### 7.1 evaluate.py — 数据管道（非 LLM 评估！）

**文件**: `datus/agent/evaluate.py` (75 行)

核心发现：`evaluate.py` **不是 LLM-as-Judge**，而是**节点间的数据管道工**。

```python
# evaluate.py:46-74
def evaluate_result(node: Node, workflow: Workflow) -> Dict:
    """Evaluate the result of a node execution and setup input for the next node."""
    try:
        # ① 把当前节点的产出写入 workflow 全局上下文
        update_result = update_context_from_node(node, workflow)
        if not update_result["success"]:
            logger.warning(f"Failed to update context from node {node.id}")

        # ② 从 workflow 全局上下文读数据，填充下一个节点的输入
        next_node = workflow.get_next_node()
        if next_node:
            return setup_node_input(next_node, workflow)
        else:
            return {"success": True, "message": "Last node, finished"}
    except Exception as e:
        return {"success": False, "message": f"Evaluation failed: {str(e)}"}
```

它只做两件事：**写上下文 → 读上下文**。没有 LLM 调用。

### 7.2 步骤①：`update_context_from_node` — 当前节点产出 → 全局上下文

以 `GenSQLAgenticNode.update_context()` 为例：

```python
# gen_sql_agentic_node.py:774-806
def update_context(self, workflow: Workflow) -> dict:
    if not self.result:
        return {"success": False, "message": "No result to update context"}

    result = self.result
    try:
        if hasattr(result, "sql") and result.sql:
            sql_result = ""
            if hasattr(result, "response") and result.response:
                _, sql_result = self._extract_sql_and_output_from_response(
                    {"content": result.response})
            new_record = SQLContext(
                sql_query=result.sql,
                explanation=result.response if hasattr(result, "response") else "",
                sql_return=sql_result,
            )
            workflow.context.sql_contexts.append(new_record)
        return {"success": True, "message": "Updated SQL generation context"}
    except Exception as e:
        return {"success": False, "message": str(e)}
```

### 7.3 步骤②：`setup_node_input` — 全局上下文 → 下一个节点的输入

以 `GenSQLAgenticNode.setup_input()` 为例：

```python
# gen_sql_agentic_node.py:141-180
def setup_input(self, workflow: Workflow) -> dict:
    task_database = workflow.task.database_name
    if task_database and self.db_func_tool:
        self._update_database_connection(task_database)

    plan_mode = workflow.metadata.get("plan_mode", False)
    auto_execute_plan = workflow.metadata.get("auto_execute_plan", False)

    if not self.input or not isinstance(self.input, GenSQLNodeInput):
        self.input = GenSQLNodeInput(
            user_message=workflow.task.task,
            external_knowledge=workflow.task.external_knowledge,
            catalog=workflow.task.catalog_name,
            database=workflow.task.database_name,
            db_schema=workflow.task.schema_name,
            schemas=workflow.context.table_schemas,    # ← schema_linking 写入的！
            metrics=workflow.context.metrics,           # ← search_metrics 写入的！
            reference_date=workflow.task.current_date,
            plan_mode=plan_mode,
            auto_execute_plan=auto_execute_plan,
        )
    return {"success": True}
```

### 7.4 workflow.context 全局数据池

```
                    workflow.context
                 ┌─────────────────────┐
                 │ table_schemas       │ ← schema_linking 写入
                 │ metrics             │ ← search_metrics 写入
                 │ sql_contexts        │ ← gen_sql 写入
                 │ execution_results   │ ← execute_sql 写入
                 │ reflections         │ ← reflect 写入
                 │ ...                 │
                 └─────────────────────┘
                        ▲       │
                        │       │
              update_context    setup_input
              (写)              (读)
```

### 7.5 完整数据流示例（reflection 工作流）

```
  ① schema_linking 节点执行完毕
     │
     ├─ update_context: workflow.context.table_schemas = [tables, columns...]
     │                                             写入
     ▼ evaluate_result()
     │
     ├─ setup_input(next=gen_sql):
     │    gen_sql.input.schemas = workflow.context.table_schemas  ← 读到 schema
     │
  ② gen_sql 节点执行完毕
     │
     ├─ update_context: workflow.context.sql_contexts = [SQLContext(...)]
     │                                             写入
     ▼ evaluate_result()
     │
     ├─ setup_input(next=execute_sql):
     │    execute_sql.input.sql = workflow.context.sql_contexts[-1].sql_query  ← 读到 SQL
     │
  ③ execute_sql 节点执行完毕
     │
     ├─ update_context: workflow.context.execution_results = [...]
     │                                             写入
     ▼ evaluate_result()
     │
     ├─ setup_input(next=reflect):
     │    reflect.input.context = workflow.context  ← 读到全部上下文
     │
  ④ reflect 节点执行完毕
     │  (如果结果不好 → 回退到 gen_sql)
     │
     ▼ evaluate_result()
     │
     ├─ setup_input(next=output):
     │    output.input = ...  ← 读到最终结果
     │
  ⑤ output 节点执行完毕
     │
     ▼ evaluate_result():
        no next node → {"success": True, "message": "Last node, finished"}
```

### 7.6 真正的"评估/裁判"在哪里？

| 位置 | 机制 | 作用 |
|---|---|---|
| **`evaluate.py`** | 数据管道 | 把节点 N 的输出传给节点 N+1，**不做质量判断** |
| **`reflect_node.py`** | LLM 审视 | 读 SQL + 执行结果，判断是否需要回退重试 |
| **`retry_policy.py`** | 规则引擎 | `GenSQLAgenticNode` 的 SQL 校验重试 |
| **`validation/`** | 校验钩子 | `builtin_checks.py` + `llm_runner.py` (LLM-as-Judge) |

---

## 八、OpenAI Agents SDK + LiteLLM 双 SDK 架构

### 8.1 一句话总结

**OpenAI Agents SDK** 负责 Agent 的"大脑循环"（多轮工具调用、会话记忆、MCP 集成），**LiteLLM** 负责"拨号上网"（统一 100+ 个 LLM 供应商的 HTTP 调用）。一个管逻辑，一个管传输。

### 8.2 分工对比

```
┌─────────────────────────────────────────────────────────────┐
│                   OpenAI Agents SDK                          │
│                    (Agentic 逻辑)                             │
│                                                             │
│  • Agent + Runner          多轮工具调用循环                    │
│  • AdvancedSQLiteSession   对话记忆持久化 (SQLite)            │
│  • Tool / FunctionTool     工具定义标准                       │
│  • MCPServerStdio          MCP 协议集成                       │
│  • RunHooks / AgentHooks   钩子系统 (权限/Token/压缩)          │
│  • ModelSettings           温度/top_p/max_tokens 配置         │
│  • RunConfig               运行时配置 + 中间输入注入           │
│  • AgentOutputSchema       结构化输出                         │
│  • 追踪系统                 span / trace 埋点                 │
│  • MaxTurnsExceeded        异常体系                           │
└─────────────────────────────────────────────────────────────┘
                              │ 内部调用
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                       LiteLLM                                │
│                    (HTTP 传输层)                              │
│                                                             │
│  • litellm.acompletion()   统一的异步 LLM 调用                │
│  • 供应商路由               anthropic/ deepseek/ gemini/...   │
│  • 参数翻译                 每个供应商的参数格式映射           │
│  • Token 计数               本地 tokenizer                    │
│  • 成本数据                 内嵌 model_prices 表              │
│  • supports_reasoning()    供应商能力探测                     │
│  • prompt caching           Anthropic cache_control 注入      │
└─────────────────────────────────────────────────────────────┘
```

### 8.3 SDK 组件在项目中的具体落脚点

| SDK 组件 | 落脚文件 | 做了什么 |
|---|---|---|
| `Agent` | `openai_compatible.py:1016` | 每个 LLM 调用前构建，配置 model/tools/mcp/hooks/settings |
| `Runner` | `openai_compatible.py:1175,1311` | `Runner.run()` 同步 + `Runner.run_streamed()` 流式 |
| `AdvancedSQLiteSession` | `session_manager.py` + `agentic_node.py` | 持久化对话历史到 `{session_id}.db` |
| `Tool` / `FunctionTool` | `tools/func_tool/base.py` → 所有工具文件 | 39 个 func_tool 全部是 SDK `FunctionTool` 实例 |
| `MCPServerStdio` | `tools/mcp_tools/mcp_server.py` | MCP 服务端连接管理 + 工具发现 |
| `RunHooks` | `tools/permission/permission_hooks.py` | 工具调用前的权限拦截 |
| `AgentHooks` | `agent/node/token_usage_hook.py` + `compact_hook.py` | Token 用量追踪 / 对话压缩 |
| `ModelSettings` | `openai_compatible.py:1016` | 从 `providers.yml` 读取温度/top_p/max_tokens |
| `RunConfig.filter` | `openai_compatible.py:947` | 交互模式下中途注入用户消息 |
| `AgentOutputSchema` | `openai_compatible.py` | 强制 LLM 按 Pydantic Schema 输出 JSON |
| `set_trace_processors` | `observability/openai_agents.py` | 接入 OpenInference 追踪 → OTLP/Langfuse |

### 8.4 核心桥接: `LiteLLMAdapter.get_agents_sdk_model()`

**文件**: `datus/models/litellm_adapter.py:384`

```python
def get_agents_sdk_model(self):
    if 供应商 == OpenAI 官方:
        return OpenAIResponsesModel(AsyncOpenAI(...))     # 直连，不经过 LiteLLM

    elif 供应商 == Anthropic Claude:
        return CacheControlLitellmModel(...)              # LiteLLM + Anthropic 缓存标记

    else:  # DeepSeek, Qwen, Kimi, Gemini, GLM, MiniMax, OpenRouter...
        return LitellmModel(...)                          # SDK 调用 LiteLLM
```

三个例外路径：

| 路径 | 何时启用 | 传输方式 |
|---|---|---|
| OpenAI 直连 | 供应商检测为 `openai` | SDK `OpenAIResponsesModel`（不经过 LiteLLM） |
| Claude 原生 | OAuth 或 vendor-native web tools | `ClaudeModel` 直接调 Anthropic Messages API |
| 标准路径 | 其他所有供应商 | SDK → `LitellmModel` → `litellm.acompletion` |

### 8.5 完整数据流追踪

```
AgenticNode._stream_once()                         agentic_node.py:3179
  │
  └─ model.generate_with_tools_stream(...)          base.py:185 (抽象契约)
       │
       └─ OpenAICompatibleModel._generate_with_tools_stream_internal
                                                    openai_compatible.py:1261
            │
            ├─ ① _build_agent():                    openai_compatible.py:1016
            │     agent = Agent(
            │       model = litellm_adapter.get_agents_sdk_model(),
            │              ↑ 三选一：OpenAIResponsesModel | CacheControlLitellmModel | LitellmModel
            │       tools = [...],                   ← func_tool 全部工具
            │       mcp_servers = {...},             ← 外部 MCP 服务
            │       hooks = permission + compact + token_usage,
            │       model_settings = ModelSettings(temperature=..., max_tokens=...),
            │     )
            │
            ├─ ② _build_run_config():               openai_compatible.py:947
            │     config = RunConfig(
            │       call_model_input_filter = lambda:  # ← 运行时注入用户新消息
            │         pending_input_queue → session → interaction_broker
            │     )
            │
            ├─ ③ Runner.run_streamed(...)             openai_compatible.py:1311
            │     │
            │     │  SDK 内部循环: 推理 → 工具调用 → 推理 → ...
            │     │    └─ LitellmModel._fetch_response()
            │     │         └─ litellm.acompletion()  ← 经 sdk_patches 包装过
            │     │
            │     └─ result.stream_events() → 逐个事件
            │
            └─ ④ 事件翻译循环:                       openai_compatible.py:1311+
                  raw_response_event  → ActionHistory(ASSISTANT)
                  run_item_stream_event → ActionHistory(TOOL)
                  result.usage          → token_usage action
                  session.store_run_usage(result)
```

### 8.6 `sdk_patches.py` — 不得不做的猴子补丁

**文件**: `datus/models/sdk_patches.py`

原因：部分国产模型（Kimi/Moonshot/DeepSeek）API 返回 `reasoning_content` 字段（思维链），但 OpenAI Agents SDK 的 `chatcmpl_converter.Converter` 不认识这个字段，在多轮工具调用时会**丢弃推理内容**，导致下一轮 LLM 失去上下文。

补丁做了三件事：

```
① 拦截 SDK 的 Converter.items_to_messages
   → 在工具调用轮次间保留/恢复 reasoning_content

② 包装 litellm.acompletion 和 litellm.completion
   → 从响应中提取 reasoning_content，缓存为 fallback
   → 流式响应用 _ReasoningContentStreamWrapper 包裹

③ 修复 Usage 序列化
   → litellm.types.utils.Usage server_tool_use 字段兼容
   → 重定向 Pydantic 序列化警告到日志
```

### 8.7 LiteLLM 的直接调用（绕过 SDK）

| 调用点 | 用途 |
|---|---|
| `openai_compatible.py:668` | **同步单次生成** `litellm.completion()` — JSON 模式/压缩调用 |
| `openai_compatible.py:1962` + `compress_utils.py:259` | **Token 计数** `litellm.token_counter` — 计算 prompt 长度 |
| `cli/effort_commands.py:208` | **能力探测** `litellm.supports_reasoning()` — `/effort` 命令 |
| `datus/__init__.py:15` | **成本地图开关** `LITELLM_LOCAL_MODEL_COST_MAP = True` — 避免 import 时网络请求 |

### 8.8 关键文件职责一览

| 文件 | 职责 |
|---|---|
| `models/base.py` | `LLMBaseModel` 抽象契约 + 工厂注册 |
| `models/openai_compatible.py` (105KB) | 10 个供应商共享引擎：构建 Agent → Runner → 事件翻译 |
| `models/litellm_adapter.py` | 供应商名称标准化 + `get_agents_sdk_model()` 三选一路由 |
| `models/litellm_cache_control.py` | 给 LiteLLM 路径注入 Anthropic prompt cache 标记 |
| `models/sdk_patches.py` | 修复 `reasoning_content` 在 SDK↔LiteLLM 边界的丢失 |
| `models/session_manager.py` (82KB) | SQLite 会话管理：checkpoint/rollback/usage 追踪 |
| `models/claude_model.py` (103KB) | Claude 原生路径：直接调 Anthropic Messages API |
| `models/codex_model.py` | Codex 模型：基于 `OpenAIResponsesModel` 的独立实现 |

---

## 九、ReAct Agent Loop 实现

### 9.1 两层实现

| 层级 | 位置 | 职责 |
|---|---|---|
| ReAct 引擎 | OpenAI Agents SDK `Runner` 内部 | Think → Act → Observe 循环（核心循环逻辑） |
| 事件翻译 | `datus/models/openai_compatible.py:1350-1650` | 将 SDK 事件流翻译为 `ActionHistory` |

### 9.2 启动 SDK 循环

```python
# openai_compatible.py:1311
result = Runner.run_streamed(
    agent,
    input=prompt,
    max_turns=max_turns,      # 最多几轮 Think-Act
    session=session,          # 多轮对话记忆 (AdvancedSQLiteSession)
    run_config=run_config,
)
```

### 9.3 SDK 内部的 ReAct 周期（伪代码）

```python
for turn in range(max_turns):
    # ── THINK ──
    response = model.fetch_response(messages, tools)
    # LLM 返回: text "我来查一下..." 或 function_call("execute_sql", ...)

    if response has no tool_calls:
        break                # LLM 认为任务完成，输出最终答案

    # ── ACT ──
    for each tool_call in response.tool_calls:
        result = execute_tool(tool_call)  # 真正执行 SQL/Bash/搜索

    # ── OBSERVE ──
    messages.append({
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": result
    })

    # → 回到 THINK，LLM 看到工具结果后决定下一步
```

### 9.4 事件翻译（Datus 包装层）

```python
# openai_compatible.py:1350-1650
while not result.is_complete:                          # SDK 的 ReAct 循环还在跑
    if interrupt_controller and interrupt_controller.is_interrupted:
        raise ExecutionInterrupted("Interrupted by user")  # ← Datus 添加的中断检查

    async for event in _stream_events_with_trace_baggage(result, agent_name):

        # ═══════════ THINK ═══════════
        if event.type == "raw_response_event":
            raw_data = event.data

            # response.output_text.delta: 逐 token 思考流
            if raw_data.type == "response.output_text.delta":
                delta_text = raw_data.delta
                yield ActionHistory(
                    role=ASSISTANT, action_type="thinking_delta",
                    output={"delta": delta_text, "accumulated": thinking_accumulated},
                    status=PROCESSING,
                )

            # response.content_part.done: 一段思考完成
            if raw_data.type == "response.content_part.done":
                full_text = strip_litellm_placeholder(thinking_accumulated.strip())
                is_thinking = len(temp_tool_calls) > 0  # 已有工具调用 = 这是思考
                yield ActionHistory(
                    role=ASSISTANT, action_type="response",
                    output={"raw_output": full_text, "is_thinking": is_thinking},
                    status=SUCCESS,
                )

        # ═══════════ ACT ═══════════
        if event.type == "run_item_stream_event":
            if event.item.type == "tool_call_item":
                tool_call_seen = True
                tool_name = raw_item.name
                arguments = raw_item.arguments
                call_id = raw_item.call_id

                temp_tool_calls[call_id] = {
                    "tool_name": tool_name, "arguments": arguments,
                    "start_time": datetime.now(),
                }
                yield ActionHistory(
                    role=TOOL, action_type=tool_name,
                    input={"function_name": tool_name, "arguments": arguments},
                    status=PROCESSING,                      # ← 执行中...
                )

            # ═══════════ OBSERVE ═══════════
            elif event.item.type == "tool_call_output_item":
                tool_output_seen = True
                output_content = event.item.output
                call_id = raw_item.call_id
                tool_info = temp_tool_calls[call_id]

                # 格式化结果摘要
                result_summary = self._format_tool_result(output_content, tool_name)
                tool_failed = _detect_tool_failure(output_content)

                yield ActionHistory(
                    role=TOOL, action_type=tool_name,
                    output={
                        "success": not tool_failed,
                        "raw_output": output_content,
                        "summary": result_summary,
                    },
                    status=FAILED if tool_failed else SUCCESS,  # ← 完成！
                )
                del temp_tool_calls[call_id]
```

### 9.5 CLI 中用户看到的 ReAct 节奏

```
Turn 1:
  THINK    raw_response_event                🤖 我来分析一下表结构...
  ACT      tool_call_item                    🔧 execute_sql("SELECT * FROM sales")
  OBSERVE  tool_call_output_item             ✓ 返回 1500 行

Turn 2:
  THINK    raw_response_event                🤖 数据拿到了，按 region 汇总...
  ACT      tool_call_item                    🔧 execute_sql("SELECT region, SUM(amount)...")
  OBSERVE  tool_call_output_item             ✓ 返回 5 行

Turn 3:
  THINK    raw_response_event                🤖 总结一下：华东区销售额最高...
  (no more tool_calls → loop ends)

                    ↓
            final message_output_item        ✅ 最终答案输出
```

### 9.6 Datus 在 SDK 循环之上的增强

| 能力 | 实现 | 位置 |
|---|---|---|
| 事件翻译 | SDK event → `ActionHistory` 统一格式 | `openai_compatible.py:1350` |
| 中断控制 | `interrupt_controller.is_interrupted` → `ExecutionInterrupted` | `openai_compatible.py:1351` |
| 异常包装 | `MaxTurnsExceeded` → `DatusException` → 压缩对话 → 自动重试一次 | `openai_compatible.py:1182-1216` |
| 工具增强 | 39 个自定义 `FunctionTool` 注入 SDK Agent | `tools/func_tool/` |
| 钩子系统 | 权限检查 + Token 追踪 + 对话压缩（通过 SDK `RunHooks`） | `agentic_node.py:3970` |
| 流式思考 | `thinking_delta` 实时输出 + `is_thinking` 标记 | `openai_compatible.py:1371-1416` |
| 占位符过滤 | `LitellmPlaceholderStreamFilter` 去掉 LiteLLM 注入的标记 | `openai_compatible.py:1342` |

**一句话**: ReAct 引擎是 SDK 的 `Runner`（`openai_compatible.py:1311` 启动），Datus 负责事件翻译、中断控制、异常恢复、工具和钩子注入。Datus 没有自己写 Think-Act-Observe 循环。

---

## 十、完整数据流全景

```
┌──────────────────────────────────────────────────────────────────────┐
│                        USER INPUT                                     │
│                   "show me sales by region"                            │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│ Agent.run(SqlTask)                           agent.py:179             │
│   → WorkflowRunner.run()                     agent.py:190             │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │
               ┌───────────────────┴───────────────────┐
               ▼                                       ▼
┌──────────────────────────┐          ┌──────────────────────────────┐
│ PLAN                     │          │ EXECUTE & REFLECT             │
│ plan.py:210              │          │ workflow_runner.py:184        │
│                          │          │                               │
│ generate_workflow()      │          │ while not complete:            │
│   → 读 workflow.yml      │          │   node.run()                  │
│   → Node.new_instance()  │          │     → AgenticNode.execute()   │
│   → 返回 Workflow DAG    │          │       → execute_stream()      │
│                          │          │         → _stream_once()      │
│                          │          │           → model.generate_   │
│                          │          │             with_tools_stream()│
│                          │          │                               │
│                          │          │   evaluate_result()            │
│                          │          │     → update_context (写)      │
│                          │          │     → setup_input (读)         │
│                          │          │   advance_to_next_node()       │
└──────────────────────────┘          └──────────────┬───────────────┘
                                                     │
                                                     ▼
┌──────────────────────────────────────────────────────────────────────┐
│               model.generate_with_tools_stream()                       │
│               openai_compatible.py:1261                                │
│                                                                        │
│ ① LiteLLMAdapter.get_agents_sdk_model()                                │
│    → OpenAIResponsesModel | CacheControlLitellmModel | LitellmModel    │
│                                                                        │
│ ② Agent(model=..., tools=[39], mcp={...}, hooks=..., settings=...)     │
│                                                                        │
│ ③ Runner.run_streamed(agent, input, max_turns, session)                │
│    ┌───────────────────────────────────────────────────────────┐      │
│    │                 SDK INTERNAL REACT LOOP                    │      │
│    │  THINK: LLM 生成文本 / 决定调工具                           │      │
│    │    ↓                                                       │      │
│    │  ACT:  执行工具 (SQL / Bash / Web / MCP)                   │      │
│    │    ↓                                                       │      │
│    │  OBSERVE: 工具结果注入对话历史                              │      │
│    │    ↓                                                       │      │
│    │  → 循环直到 LLM 认为任务完成                                │      │
│    └───────────────────────────────────────────────────────────┘      │
│                                                                        │
│ ④ 事件翻译:                                                            │
│    raw_response_event → ActionHistory(ASSISTANT)                       │
│    tool_call_item     → ActionHistory(TOOL, PROCESSING)                │
│    tool_call_output   → ActionHistory(TOOL, SUCCESS/FAILED)            │
│    usage              → token_usage action                             │
└──────────────────────────────────┬───────────────────────────────────┘
                                   │
                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     FINAL RESULT                                       │
│  _build_success_result(ctx) → result_class.model_validate()            │
│  Workflow.save(trajectory.yaml)                                       │
│  return final_result                                                   │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 附录：关键文件索引

| 文件 | 行数 | 对应章节 |
|---|---|---|
| `datus/agent/agent.py` | 1256 | 二 |
| `datus/agent/workflow_runner.py` | 380 | 三 |
| `datus/agent/node/agentic_node.py` | 4082 | 四 |
| `datus/agent/plan.py` | 278 | 六 |
| `datus/agent/workflow.yml` | 43 | 六 |
| `datus/agent/evaluate.py` | 75 | 七 |
| `datus/agent/node/gen_sql_agentic_node.py` | ~900 | 四、七 |
| `datus/models/base.py` | ~200 | 八 |
| `datus/models/openai_compatible.py` | ~2000 | 八、九 |
| `datus/models/litellm_adapter.py` | ~550 | 八 |
| `datus/models/litellm_cache_control.py` | ~130 | 八 |
| `datus/models/sdk_patches.py` | ~800 | 八 |
| `datus/models/session_manager.py` | ~2000 | 八 |
| `datus/models/claude_model.py` | ~2500 | 八 |
| `datus/tools/func_tool/plan_tools.py` | ~600 | 六 |
| `datus/agent/node/chat_agentic_node.py` | ~600 | 六 |
