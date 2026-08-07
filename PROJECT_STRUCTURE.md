# Datus-Agent 项目完整目录结构分析

> **项目概述**：datus-agent v0.3.9 — AI 驱动的数据分析 Agent，实现 NL→SQL 转换、多数据库支持、RAG 知识库、MCP 协议。
> **技术栈**：Python 3.12+、OpenAI Agents SDK + LiteLLM、LanceDB、FastAPI、FastMCP、Streamlit
> **包管理器**：uv · **许可证**：Apache-2.0

---

## 一、根目录

| 目录/文件 | 类型 | 用途 | 关键文件说明 |
|---|---|---|---|
| `datus/` | **核心** | 主源码包 | 所有业务逻辑，约 500+ 文件 |
| `tests/` | **测试** | 测试套件 | unit_tests 1:1 镜像源码结构，integration/regression 端到端测试 |
| `ci/` | **构建** | CI 流水线 | PR/Nightly 测试编排，覆盖率审查，发布就绪检查 |
| `.github/` | **配置** | GitHub CI/CD | 15 个 workflows + Issue/PR 模板 + Mergify 自动合并 |
| `docs/` | **文档** | MkDocs 文档站点 | 中英双语，覆盖 CLI/API/配置/工作流等 20+ 主题 |
| `conf/` | **配置** | 部署配置样例 | `agent.yml.example`、`providers.yml`、`auth_clients.yml.example` |
| `benchmark/` | **测试** | 基准测试框架 | BIRD/Spider2/Semantic Layer 评估脚本 |
| `build_scripts/` | **构建** | 打包与容器 | `build_pypi_package.py`、`Dockerfile`、`build_test_data.sh` |
| `scripts/` | **辅助** | 临时开发脚本 | Spark 测试、订阅调试、回归运行 |
| `subject/` | **配置** | 示例语义模型 | `california_schools/` 域（frpm/satscores/schools 三个 YAML） |
| `pyproject.toml` | **配置** | 项目元数据 | 依赖、ruff、pytest、coverage 配置 |
| `mkdocs.yml` | **配置** | 文档站点配置 | MkDocs 导航与主题 |
| `Makefile` | **构建** | 构建入口 | 封装 clean/build/install/upload/publish |
| `uv.lock` | **配置** | 锁定依赖 | 规范依赖锁定文件 |
| `CLAUDE.md` | **配置** | 项目指令 | Claude Code 的项目级行为规范 |
| `.gitmodules` | **配置** | 子模块声明 | `benchmark/spider2` → xlang-ai/Spider2 |
| `.pre-commit-config.yaml` | **配置** | Pre-commit 钩子 | 提交前自动检查 |

---

## 二、`.github/` — CI/CD 基础设施

| 子目录/文件 | 类型 | 用途 | 关键文件说明 |
|---|---|---|---|
| `workflows/` | **构建** | 15 条流水线 | `code-quality.yml` 代码质量、`merge-queue.yml` 合并队列、`run-nightly.yml` 夜间测试、`prepare-release.yml` 发布准备、`deploy-docs.yml` 文档部署、`run-ut-and-coverage.yml` 单元测试覆盖率、`python-format-check.yml` 格式检查、`title-check.yml` PR 标题检查、`publish-release.yml` 正式发布、`release-candidate.yml` 候选发布、`release-branch-guard.yml` 分支保护、`trigger-backport-on-merge.yml` 回移植触发、`sync-backport-checklist.yml` 同步回移植清单、`sync-release-blocker.yml` 同步阻塞项、`test-audit.yml` 测试审计 |
| `ISSUE_TEMPLATE/` | **配置** | Issue 模板 | `bug_report.yml`、`feature_request.yml`、`other.yml`、`config.yml` |
| `mergify.yml` | **配置** | 自动合并 | Mergify 规则配置 |
| `PULL_REQUEST_TEMPLATE.md` | **配置** | PR 模板 | 标准三段式（Why / Solution / Test Cases） |
| `pr-title-checker-config.json` | **配置** | PR 标题检查 | 正则规则配置（如 `[BugFix]` 前缀） |

---

## 三、`datus/` — 主源码包（逐层展开）

### 3.1 `datus/` 根层

| 文件 | 类型 | 用途 |
|---|---|---|
| `__init__.py` | 核心 | 包初始化，设置 `LITELLM_LOCAL_MODEL_COST_MAP`，版本检测 |
| `entrypoints.py` | 核心 | 五个控制台入口点：`datus`、`datus-cli`、`datus-api`、`datus-mcp`、`datus-gateway`，含多进程策略配置 |
| `main.py` (23KB) | 核心 | Agent 模式主启动器 |
| `mcp_server.py` (50KB) | 核心 | MCP 服务端实现，对外暴露所有工具能力 |
| `multi_round_benchmark.py` | 辅助 | 多轮对话基准测试运行器 |
| `conf/providers.yml` | 配置 | LLM 供应商定义（与根 `conf/providers.yml` 同步） |
| `resources/skills/` | 配置 | 23 个内置技能包 |
| `sample_data/` | 配置 | 示例数据集 |

---

### 3.2 `datus/agent/` — 工作流编排引擎（核心中的核心）

| 文件/子目录 | 类型 | 用途 | 关键文件说明 |
|---|---|---|---|
| `agent.py` (58KB) | **核心** | Agent 主类 | plan → execute → reflect 循环，核心调度逻辑 |
| `workflow.py` | **核心** | 工作流定义 | 工作流模型与生命周期管理 |
| `workflow.yml` | **核心** | 默认工作流配置 | 节点编排 DAG 定义 |
| `workflow_runner.py` | **核心** | 工作流执行器 | 按 DAG 顺序执行各节点，管理数据传递 |
| `plan.py` | **核心** | 计划模式 | 自然语言 → 任务分解 → 执行计划 |
| `evaluate.py` | **核心** | 评估框架 | 输出质量评估与打分 |
| `reflect.py` | **核心** | 反思循环 | 结果反思、错误分析、自我修正 |
| `node/` | **核心** | 49 个节点实现 | 所有 agentic node（见下方展开） |

#### `node/` 子目录展开：

| 分类 | 关键文件 | 用途 |
|---|---|---|
| **核心引擎** | `agentic_node.py` (197KB!) | 通用 agentic node 引擎：LLM 轮次循环、工具调度、记忆管理、子 agent 编排 |
| **基类** | `node.py`、`node_factory.py` | Node 抽象基类 + 工厂模式构建（从配置创建节点实例） |
| **控制流节点** | `parallel_node.py` | 并行执行多个子节点 |
| | `subworkflow_node.py` | 嵌套子工作流 |
| | `selection_node.py` | 条件路由 / 分支选择 |
| | `output_node.py` | 结果输出与格式化 |
| | `begin_node.py` | 工作流入口节点 |
| | `hitl_node.py` | Human-in-the-Loop 人在回路 |
| **领域问答节点** | `ask_metrics_agentic_node.py` | 指标问答（自然语言查指标） |
| | `ask_dashboard_agentic_node.py` | 仪表板问答 |
| | `ask_report_agentic_node.py` | 报表问答 |
| | `chat_agentic_node.py` | 通用对话节点 |
| | `explore_agentic_node.py` | 数据探索节点 |
| | `compare_agentic_node.py` / `compare_node.py` | 数据对比分析 |
| | `feedback_agentic_node.py` | 用户反馈收集与学习 |
| | `sql_summary_agentic_node.py` | SQL 查询摘要 |
| | `scheduler_agentic_node.py` | 定时调度任务 |
| **生成节点** | `gen_sql_agentic_node.py` | NL → SQL 生成 |
| | `gen_metrics_agentic_node.py` | 指标定义生成 |
| | `gen_semantic_model_agentic_node.py` | 语义模型自动生成 |
| | `gen_dashboard_agentic_node.py` | 仪表板生成 |
| | `gen_report_agentic_node.py` | 报表生成 |
| | `gen_visual_dashboard_agentic_node.py` | 可视化仪表板生成（HTML） |
| | `gen_visual_report_agentic_node.py` | 可视化报表生成（HTML） |
| | `gen_table_agentic_node.py` | 表结构生成 |
| | `gen_skill_agentic_node.py` | 技能/工作流生成 |
| | `gen_job_agentic_node.py` | 定时任务生成 |
| **分析节点** | `execute_sql_node.py` | SQL 执行（调用 DB 连接器） |
| | `reason_sql_node.py` | SQL 结果推理/解释 |
| | `schema_linking_node.py` | Schema 链接（NL 实体 → 表/列） |
| | `search_metrics_node.py` | 指标搜索（向量 + 关键词） |
| | `date_parser_node.py` | 日期表达式解析 |
| | `doc_search_node.py` | 文档知识库搜索 |
| | `fix_node.py` | SQL 错误自动修复 |
| | `reflect_node.py` | 结果反思节点 |
| | `deliverable_node.py` | 交付物整理 |
| | `semantic_authoring.py` | 语义模型编辑/创作 |
| **记忆/压缩** | `compact_hook.py` | 对话上下文压缩钩子 |
| | `compact_archive.py` | 压缩历史归档 |
| | `compact_prompts.py` | 压缩提示词模板 |
| | `token_usage_hook.py` | Token 用量追踪与告警 |
| | `stream_run_context.py` | 流式运行上下文管理 |
| | `retry_policy.py` | LLM 调用重试策略 |
| **可视化物** | `base_artifact_ask_agentic_node.py` | 可视化物问答基类 |
| | `base_visual_artifact_agentic_node.py` | 可视化物生成基类 |
| | `visual_artifact/dashboard_html_renderer.py` | 仪表板 HTML 渲染器 |
| | `visual_artifact/report_html_renderer.py` | 报表 HTML 渲染器 |
| | `visual_artifact/_artifact_html_renderer.py` | 通用 HTML 物渲染 |
| | `visual_artifact/_visual_artifact_finalize.py` | 可视化物最终化处理 |
| | `visual_artifact/templates/` | Jinja2 HTML 模板 |

---

### 3.3 `datus/api/` — FastAPI 后端服务（核心）

| 子目录/文件 | 类型 | 用途 | 关键文件说明 |
|---|---|---|---|
| `main.py` | **核心** | FastAPI 应用组装 | 路由注册、中间件、生命周期事件 |
| `service.py` | **核心** | 服务门面 | 统一服务入口，协调所有子服务 |
| `deps.py` | **核心** | 依赖注入 | FastAPI DI 依赖（Agent 实例、配置等） |
| `constants.py` | **核心** | API 常量 | 端点路径、状态码等常量 |
| `legacy_auth.py` | 辅助 | 向后兼容认证 | 旧版认证兼容层 |
| `legacy_models.py` | 辅助 | 向后兼容模型 | 旧版数据模型兼容层 |

| 子目录 | 类型 | 关键文件 | 用途 |
|---|---|---|---|
| `auth/` | **核心** | `provider.py`（认证提供者接口）、`no_auth_provider.py`（无认证）、`loader.py`（认证加载器）、`context.py`（认证上下文） | 可插拔认证系统 |
| `hooks/` | **核心** | `chat_hooks.py`（对话生命周期钩子）、`metric_hooks.py`（指标钩子） | 请求/响应拦截与处理 |
| `models/` (15 文件) | **核心** | `chat_models.py`（对话请求/响应）、`agent_models.py`（Agent 配置）、`config_models.py`（系统配置）、`database_models.py`（数据库连接）、`dashboard_models.py`（仪表板）、`report_models.py`（报表）、`kb_models.py`（知识库）、`mcp_models.py`（MCP 配置）、`table_models.py`（表结构）、`visualization_models.py`（可视化）、`explorer_models.py`（探索）、`success_story_models.py`（成功案例）、`cli_models.py`（CLI 模型）、`base_models.py`（基础模型） | Pydantic 请求/响应 Schema |
| `routes/` (15 文件) | **核心** | 与 models 一一对应：`chat_routes.py`、`agent_routes.py`、`config_routes.py`、`database_routes.py`、`dashboard_routes.py`、`report_routes.py`、`kb_routes.py`、`mcp_routes.py`、`table_routes.py`、`visualization_routes.py`、`explorer_routes.py`、`success_story_routes.py`、`tool_routes.py`、`cli_routes.py`、`models_routes.py` | FastAPI 路由定义 |
| `services/` (17 文件) | **核心** | `agent_service.py`（Agent 编排）、`chat_service.py`（对话逻辑）、`chat_task_manager.py`（对话任务调度）、`database_service.py`（数据库操作）、`dashboard_service.py`（仪表板逻辑）、`report_service.py`（报表逻辑）、`kb_service.py`（知识库管理）、`mcp_service.py`（MCP 集成）、`cli_service.py`（CLI 服务）、`tool_service.py`（工具调度）、`visualization_service.py`（可视化）、`explorer_service.py`（数据探索）、`success_story_service.py`（成功案例）、`datus_service.py`（主服务门面）、`datus_service_cache.py`（服务缓存）、`background_drain.py`（后台任务排出）、`action_sse_converter.py`（SSE 事件转换） | 业务逻辑层 |
| `utils/` | **核心** | `path_utils.py`（路径处理）、`semantic_validation.py`（语义校验）、`stream_cancellation.py`（流取消）、`stream_errors.py`（流错误处理） | API 专用工具 |

---

### 3.4 `datus/cli/` — 交互式命令行/TUI（核心，84 个条目）

| 分类 | 关键文件 | 用途 |
|---|---|---|
| **主入口** | `main.py` | CLI 命令分发入口 |
| | `repl.py` (122KB) | 核心 REPL 循环（提示输入、命令解析、结果渲染） |
| | `agent_app.py`、`agent_commands.py` | Agent 模式交互界面 |
| **对话引擎** | `chat_commands.py` (104KB) | 对话模式全部命令实现 |
| | `autocomplete.py` (71KB) | prompt-toolkit 自动补全引擎（SQL/表名/列名/技能等） |
| **TUI 系统** | `tui/app.py` (118KB) | Textual 全屏 TUI 主体应用 |
| | `tui/output_buffer.py` | 输出缓冲管理 |
| | `tui/selection.py` | 列表选择组件 |
| | `tui/region_selection.py` | 区域选择（多选） |
| | `tui/search.py` | 搜索组件 |
| | `tui/scrollbar.py` | 自定义滚动条 |
| | `tui/clipboard.py` | 剪贴板集成 |
| | `tui/console_bridge.py` | Rich Console ↔ Textual 桥接 |
| | `tui/live_display_state.py` | 实时显示状态管理 |
| | `tui/wizard_host.py` | 向导弹窗宿主 |
| **Screen 层** | `screen/catalog_screen.py` (58KB) | 知识库目录浏览屏 |
| | `screen/subject_screen.py` (88KB) | 主题/数据源浏览屏 |
| | `screen/workflow_screen.py` | 工作流可视化屏 |
| | `screen/context_screen.py` | 上下文管理屏 |
| | `screen/context_app.py` | 上下文应用 |
| | `screen/base_app.py`、`screen/base_widgets.py` | Screen 基类与通用组件 |
| **流式渲染** | `action_display/display.py` | 动作展示调度 |
| | `action_display/streaming.py` (95KB) | 流式输出引擎（Markdown/表格/代码等） |
| | `action_display/tool_content.py` (95KB) | 工具调用内容渲染（SQL 高亮、结果表格等） |
| | `action_display/renderers.py` | 各类内容渲染器 |
| | `action_display/markdown_stream.py` | Markdown 增量流式渲染 |
| **引导向导** | `bootstrap_app.py`、`bootstrap_commands.py` | 项目初始化引导 |
| | `bootstrap_bi_app.py`、`bootstrap_bi_commands.py` | BI 工具引导（Superset/Grafana） |
| | `bootstrap_bi_picker.py`、`bootstrap_bi_streams.py`、`bootstrap_bi_subagents.py` | BI 引导子组件 |
| | `bootstrap_streams.py`、`bootstrap_subagent.py` | 流/子Agent 引导 |
| | `interaction_app.py`、`interactive_init.py` | 交互式初始化 |
| | `project_init.py`、`init_commands.py`、`init_util.py` | 项目初始化逻辑 |
| | `profile_picker_app.py` | 配置选择器 |
| | `list_selector_app.py` | 通用列表选择器 |
| | `language_app.py`、`language_commands.py` | 语言选择 |
| | `effort_app.py`、`effort_commands.py` | 推理力度选择 |
| **数据源管理** | `datasource_app.py`、`datasource_commands.py`、`datasource_manager.py` | 数据源增删改查 UI |
| **模型管理** | `model_app.py`、`model_commands.py` | LLM 模型选择与配置 UI |
| | `provider_model_catalog.py` | 供应商模型目录 |
| | `provider_auth_flows.py` | 供应商认证流程 |
| **MCP 管理** | `mcp_app.py`、`mcp_commands.py` | MCP 服务器管理 UI |
| **技能管理** | `skill_app.py`、`skill_commands.py`、`skill_cli.py`、`skill_command_utils.py` | 技能安装/卸载/配置 UI |
| **服务管理** | `service_commands.py`、`service_manager.py` | Agent 守护进程管理（启动/停止/状态） |
| | `service_client.py` | 与守护进程通信的客户端 |
| | `service_bootstrap.py` | 服务引导 |
| | `service_adapter_installer.py` | 适配器安装 |
| | `service_config_app.py` | 服务配置 UI |
| | `upgrade_service.py`、`upgrade_cli.py` | 版本升级逻辑 |
| **插件系统** | `plugin_app.py`、`plugin_cli.py`、`plugin_commands.py` | 插件安装/卸载/管理 UI |
| | `plugin_pack.py` | 插件打包工具 |
| | `plugin_service.py` | 插件服务层 |
| **执行引擎** | `bash_mode.py`、`sandbox_commands.py` | Bash 沙箱执行模式 |
| | `manual_exec.py` | 手动执行入口 |
| | `execution_state.py` | 执行状态机 |
| | `generation_hooks.py` (60KB) | 生成钩子（拦截/修改 LLM 输出） |
| | `tool_arg_parser.py` | 工具参数解析 |
| **子Agent** | `sub_agent_wizard.py` (88KB) | 子 Agent 创建/配置向导 |
| **样式系统** | `cli_styles.py` | 全局颜色/符号定义（`print_error`、`print_success` 等） |
| | `_render_utils.py` | 表格/代码渲染工具 |
| **Web 内嵌** | `web/chatbot.py` | 内嵌 Web 聊天机器人后端 |
| | `web/chat_executor.py` | 聊天执行引擎 |
| | `web/config_manager.py` | Web 配置管理 |
| | `web/templates/index.html` | 聊天界面 HTML 模板 |
| **其他** | `subject_rich_utils.py` | 主题 Rich 渲染 |
| | `metadata_commands.py` | 元数据管理命令 |
| | `context_commands.py` | 上下文管理命令 |
| | `summarize_commands.py` | 摘要命令 |
| | `build_kb_commands.py` | 知识库构建命令 |
| | `background_sync.py` | 后台同步 |
| | `bang_command.py` | `!` 前缀命令处理 |
| | `slash_registry.py` | `/` 命令注册表 |
| | `status_bar.py` | 状态栏组件 |
| | `todo_sidebar.py` | 待办侧栏 |
| | `cli_context.py` | CLI 上下文管理 |
| | `print_mode.py` | 打印模式控制 |
| | `input_modes.py` | 输入模式管理 |
| | `blocking_input_manager.py` | 阻塞输入管理 |

---

### 3.5 `datus/configuration/` — 配置系统（核心）

| 文件 | 类型 | 用途 |
|---|---|---|
| `agent_config.py` (153KB) | **核心** | 主配置 dataclass：模型定义、数据源、工作流、存储后端、基准测试、插件、调度器等全部配置项 |
| `agent_config_loader.py` | **核心** | YAML 加载 + `${ENV_VAR}` 环境变量替换 + 校验逻辑 |
| `project_config.py` | **核心** | 项目级配置读/写（`.datus/config.yml`），管理 `target`、`default_datasource` 等白名单字段 |
| `node_type.py` | **核心** | NodeType 枚举：定义所有可用节点类型的注册表 |
| `inherited_memory_overrides.py` | 辅助 | 记忆/配置继承覆盖机制 |
| `scoped_context_overrides.py` | 辅助 | 作用域上下文覆盖 |

---

### 3.6 `datus/models/` — 多 LLM 适配层（核心）

| 文件 | 类型 | 用途 |
|---|---|---|
| `base.py` | **核心** | `LLMBaseModel` 抽象基类 + `MODEL_TYPE_MAP` 工厂注册 |
| `openai_compatible.py` (105KB) | **核心** | 共享 OpenAI 兼容引擎：流式响应、工具调用、重试策略、Token 计数 |
| `claude_model.py` (103KB) | **核心** | Anthropic Claude 完整实现：Messages API、system prompt、tool use |
| `session_manager.py` (82KB) | **核心** | SQLite 会话/上下文管理：`~/.datus/sessions/{project}/{session_id}.db` |
| `litellm_adapter.py` | **核心** | LiteLLM 统一接口适配 |
| `litellm_cache_control.py` | **核心** | LiteLLM 级 prompt 缓存控制 |
| `openai_model.py` | **核心** | OpenAI 标准模型适配 |
| `codex_model.py` | **核心** | OpenAI Codex 模型适配 |
| `openrouter_model.py` | **核心** | OpenRouter 多供应商代理 |
| `deepseek_model.py` | 辅助 | DeepSeek 模型桩（复用 `openai_compatible.py`） |
| `gemini_model.py` | 辅助 | Google Gemini 模型桩 |
| `qwen_model.py` | 辅助 | 通义千问模型桩 |
| `kimi_model.py` | 辅助 | Moonshot Kimi 模型桩 |
| `glm_model.py` | 辅助 | 智谱 GLM 模型桩 |
| `minimax_model.py` | 辅助 | MiniMax 模型桩 |
| `sdk_patches.py` | 辅助 | OpenAI Agents SDK 补丁/扩展 |
| `mcp_utils.py` | 辅助 | MCP 结果处理工具 |
| `mcp_result_extractors.py` | 辅助 | MCP 结果提取器 |

---

### 3.7 `datus/gateway/` — 即时通讯网关（核心）

| 文件/子目录 | 类型 | 用途 |
|---|---|---|
| `main.py` | **核心** | 网关主入口 |
| `runtime.py` | **核心** | 网关运行时：连接管理、消息路由、心跳 |
| `bridge.py` | **核心** | 核心桥接：IM 消息 ↔ Agent 请求/响应 |
| `commands.py` | **核心** | 命令解析（`/` 命令支持） |
| `formatters.py` | **核心** | 消息格式化（IM 平台适配） |
| `models.py` | **核心** | 网关数据模型 |
| `configure.py` | **核心** | 网关配置管理 |
| `adapters/feishu.py` | **核心** | 飞书（Lark）平台适配器：Webhook、消息卡片、事件订阅 |
| `adapters/slack.py` | **核心** | Slack 平台适配器：Socket Mode、Block Kit |
| `channel/base.py` | **核心** | 通道抽象接口 |
| `channel/registry.py` | **核心** | 通道注册表 |
| `richtext/ir.py` | **核心** | 富文本中间表示（IR） |
| `richtext/parser.py` | **核心** | IR 解析器：Markdown → IR |
| `richtext/render.py` | **核心** | IR 渲染器：IR → 平台格式 |
| `richtext/chunker.py` | **核心** | 长消息分块（适配 IM 字数限制） |
| `richtext/escape.py` | **核心** | 特殊字符转义 |

---

### 3.8 `datus/storage/` — 知识库存储层（核心，LanceDB + SQLite 双引擎）

| 分类 | 关键文件 | 用途 |
|---|---|---|
| **基类与编排** | `base.py` (32KB) | `StorageBase` / `BaseEmbeddingStore` 抽象基类 |
| | `registry.py` | 存储后端注册表 |
| | `backend_holder.py` | 后端实例持有与管理 |
| | `storage_cfg.py` | 存储配置管理 |
| | `catalog_manager.py` | 知识库目录管理 |
| | `subject_manager.py` | 主题（领域）管理 |
| | `session_state.py` | 会话状态存储 |
| | `artifact_replacement.py` | 物替换逻辑 |
| | `knowledge_provenance.py` | 知识来源追踪 |
| | `datasource_scope.py` | 数据源作用域 |
| | `rag_scope.py` | RAG 搜索范围控制 |
| | `scoped_filter.py` | 作用域过滤器 |
| **向量存储** | `vector/lance_backend.py` | LanceDB 向量存储后端（嵌入式向量检索） |
| **关系型存储** | `rdb/sqlite_backend.py` | SQLite 关系存储后端 |
| **嵌入模型** | `embedding_models.py` | 嵌入模型抽象 |
| | `embedding_openai.py` | OpenAI Embedding 适配 |
| | `fastembed_embeddings.py` | FastEmbed 本地嵌入（无需 API） |
| | `embedding_diagnostics.py` | 嵌入质量诊断 |
| | `fts.py` | 全文搜索（FTS5） |
| **文档知识库** | `document/store.py` | 文档向量存储 |
| | `document/streaming_processor.py` | 文档流式摄取处理 |
| | `document/doc_init.py` | 文档知识库初始化 |
| | `document/schemas.py` | 文档 Schema 定义 |
| | `document/chunker/semantic_chunker.py` | 语义分块器 |
| | `document/cleaner/doc_cleaner.py` | 文档清洗器 |
| | `document/fetcher/base_fetcher.py` | 文档抓取基类 |
| | `document/fetcher/web_fetcher.py` | Web 页面抓取 |
| | `document/fetcher/github_fetcher.py` | GitHub 仓库抓取 |
| | `document/fetcher/local_fetcher.py` | 本地文件抓取 |
| | `document/fetcher/rate_limiter.py` | 请求频率限制 |
| | `document/nav_resolver/base_resolver.py` | 导航解析基类 |
| | `document/nav_resolver/detector.py` | 文档站点类型检测 |
| | `document/nav_resolver/docusaurus_resolver.py` | Docusaurus 站点解析 |
| | `document/nav_resolver/hugo_resolver.py` | Hugo 站点解析 |
| | `document/nav_resolver/mkdocs_resolver.py` | MkDocs 站点解析 |
| | `document/nav_resolver/fallback_resolver.py` | 通用回退解析 |
| | `document/parser/html_parser.py` | HTML → Markdown 解析 |
| | `document/parser/markdown_parser.py` | Markdown 结构化解析 |
| | `document/parser/metadata_extractor.py` | 元数据提取 |
| **语义模型** | `semantic_model/store.py` | 语义模型向量存储 |
| | `semantic_model/semantic_model_init.py` | 语义模型初始化 |
| | `semantic_model/adapter_init.py` | 适配器初始化 |
| | `semantic_model/auto_create.py` | 语义模型自动创建 |
| | `semantic_model/profile_description.py` | 表画像描述 |
| **指标** | `metric/store.py` | 指标定义存储 |
| | `metric/metric_init.py` | 指标初始化 |
| | `metric/adapter_init.py` | 指标适配器初始化 |
| | `metric/subject_path.py` | 主题路径管理 |
| **参考 SQL** | `reference_sql/store.py` | 黄金 SQL 样例存储 |
| | `reference_sql/reference_sql_init.py` | 参考 SQL 初始化 |
| | `reference_sql/sql_file_processor.py` | SQL 文件处理 |
| **参考模板** | `reference_template/store.py` | Jinja2 模板知识库 |
| | `reference_template/reference_template_init.py` | 模板初始化 |
| | `reference_template/template_file_processor.py` | 模板文件处理 |
| **Schema 元数据** | `schema_metadata/store.py` | 表/列 Schema 元数据存储 |
| | `schema_metadata/local_init.py` | 本地 Schema 初始化 |
| | `schema_metadata/benchmark_init.py` | 基准测试初始化 |
| | `schema_metadata/benchmark_init_bird.py` | BIRD 数据集初始化 |
| **其他** | `feedback/store.py` | 反馈循环存储 |
| | `kb_retrieval/store.py` | 知识库检索存储 |
| | `subject_tree/store.py` | 主题树存储 |
| | `table_semantic_profile/store.py` | 表语义画像存储 |
| | `task/store.py` | 任务存储 |
| | `storage/init_utils.py`（多子包） | 各存储模块公共初始化工具 |

---

### 3.9 `datus/tools/` — 工具层（核心，Agent 可调用的全部工具）

| 分类 | 关键文件 | 用途 |
|---|---|---|
| **基础** | `base.py` | 工具抽象基类 |
| | `tool_docstrings.py` | 为 LLM 生成工具描述文档 |
| | `sql_guard.py` | SQL 安全守护（危险操作拦截） |
| | `sql_policy.py` | SQL 执行策略（行数限制、超时等） |
| **函数工具** (39 文件) | `func_tool/base.py` | 函数工具基类 |
| | `func_tool/database.py` | 数据库查询工具（多 DB 支持） |
| | `func_tool/bash_tool.py` | Bash 命令执行 |
| | `func_tool/bash_sandbox.py` | Bash 沙箱安全执行 |
| | `func_tool/filesystem_tools.py` | 文件系统操作 |
| | `func_tool/memory_tools.py` | 记忆读写工具 |
| | `func_tool/memory_filesystem_tools.py` | 记忆文件系统 |
| | `func_tool/ask_user_tools.py` | 向用户提问工具 |
| | `func_tool/plan_tools.py` | 计划管理工具 |
| | `func_tool/orchestrator_tools.py` | 编排工具 |
| | `func_tool/scheduler_tools.py` | 定时调度工具 |
| | `func_tool/web_tool.py` | Web 搜索工具 |
| | `func_tool/bi_tools.py` | BI 工具（Superset/Grafana 集成） |
| | `func_tool/semantic_tools.py` | 语义模型工具 |
| | `func_tool/semantic_discovery_tools.py` | 语义发现工具 |
| | `func_tool/metric_filesystem_tools.py` | 指标文件系统 |
| | `func_tool/metric_queryability.py` | 指标可查询性检查 |
| | `func_tool/generation_tools.py` | 内容生成工具 |
| | `func_tool/generation_evidence.py` | 生成证据追溯 |
| | `func_tool/attribution_utils.py` | 归因工具 |
| | `func_tool/report_artifact_tools.py` | 报表物工具 |
| | `func_tool/dashboard_artifact_tools.py` | 仪表板物工具 |
| | `func_tool/reference_template_tools.py` | 参考模板工具 |
| | `func_tool/context_search.py` | 上下文搜索 |
| | `func_tool/session_search_tool.py` | 会话历史搜索 |
| | `func_tool/platform_doc_search.py` | 平台文档搜索 |
| | `func_tool/date_parsing_tools.py` | 日期解析工具 |
| | `func_tool/sub_agent_task_tool.py` | 子 Agent 任务分发 |
| | `func_tool/skill_validate_tool.py` | 技能校验工具 |
| | `func_tool/sql_modeling_planner.py` | SQL 建模规划器 |
| | `func_tool/osi_target_tools.py` | OSI 目标工具 |
| | `func_tool/fs_path_policy.py` | 文件系统路径安全策略 |
| **数据库工具** | `db_tools/db_manager.py` | 多数据库连接器管理器 |
| | `db_tools/config.py` | 数据库配置 |
| | `db_tools/builtin_configs.py` | 内建数据库配置 |
| | `db_tools/capabilities.py` | 数据库能力检测 |
| | `db_tools/dialect_config.py` | SQL 方言配置 |
| | `db_tools/duckdb_connector.py` | DuckDB 连接器 |
| | `db_tools/sqlite_connector.py` | SQLite 连接器 |
| | `db_tools/_migration_compat.py` | 迁移兼容层 |
| **LLM 工具** | `llms_tools/autofix_sql.py` | SQL 自动修复（LLM 驱动） |
| | `llms_tools/reasoning_sql.py` | SQL 推理分析 |
| | `llms_tools/match_schema.py` | Schema 匹配 |
| | `llms_tools/visualization_tool.py` | 可视化生成工具 |
| | `llms_tools/mcp_stream_utils.py` | MCP 流式工具 |
| **MCP 工具** | `mcp_tools/mcp_config.py` | MCP 配置管理 |
| | `mcp_tools/mcp_manager.py` | MCP 客户端管理器 |
| | `mcp_tools/mcp_server.py` | MCP 服务端（内部） |
| | `mcp_tools/mcp_tool.py` | MCP 工具代理 |
| **权限系统** | `permission/permission_manager.py` | 权限管理器（统一入口） |
| | `permission/permission_config.py` | 权限配置 |
| | `permission/permission_hooks.py` | 权限钩子 |
| | `permission/bash_classifier.py` | Bash 命令分类器（安全/危险） |
| | `permission/bash_rules.py` | Bash 安全规则 |
| | `permission/profiles.py` | 权限配置预案 |
| **技能工具** | `skill_tools/skill_manager.py` | 技能生命周期管理 |
| | `skill_tools/skill_registry.py` | 技能注册表 |
| | `skill_tools/skill_config.py` | 技能配置 |
| | `skill_tools/skill_bundle.py` | 技能打包 |
| | `skill_tools/skill_func_tool.py` | 技能函数工具 |
| | `skill_tools/marketplace_client.py` | 技能市场客户端 |
| | `skill_tools/marketplace_auth.py` | 技能市场认证 |
| **其他** | `bi_tools/dashboard_assembler.py` | BI 仪表板组装 |
| | `date_tools/date_parser.py` | 日期解析器 |
| | `lineage_graph_tools/schema_lineage.py` | Schema 血缘图 |
| | `middleware/tool_middleware.py` | 工具调用中间件管线 |
| | `output_tools/output.py` | 输出格式化工具 |
| | `proxy/proxy_tool.py` | 工具代理 |
| | `proxy/tool_result_channel.py` | 工具结果通道 |
| | `registry/tool_registry.py` | 工具注册表 |
| | `search_tools/search_tool.py` | 搜索工具 |
| | `semantic_tools/base.py` | 语义工具基类 |
| | `semantic_tools/config.py` | 语义工具配置 |
| | `semantic_tools/models.py` | 语义工具模型 |
| | `semantic_tools/registry.py` | 语义工具注册 |
| | `semantic_tools/storage_sync.py` | 语义工具存储同步 |

---

### 3.10 `datus/prompts/` — 提示词管理（核心）

| 文件/子目录 | 类型 | 用途 |
|---|---|---|
| `prompt_manager.py` | **核心** | Jinja2 模板加载/渲染引擎 |
| `gen_sql.py` | **核心** | NL → SQL 生成提示逻辑 |
| `compare_sql.py` | **核心** | SQL 对比提示逻辑 |
| `compare_sql_with_mcp.py` | **核心** | MCP 增强的 SQL 对比 |
| `fix_sql.py` | **核心** | SQL 修复提示逻辑 |
| `reasoning_sql_with_mcp.py` | **核心** | MCP 增强的 SQL 推理 |
| `schema_lineage.py` | **核心** | Schema 血缘推理提示 |
| `selection.py` | **核心** | 选项/工具选择提示 |
| `reflection.py` | **核心** | 反思/自我修正提示 |
| `extract_dates.py` | **核心** | 日期提取提示 |
| `database_notes.py` | **核心** | 数据库备注/上下文提示 |
| `output_checking.py` | **核心** | 输出校验提示 |
| `prompt_templates/` (60+ .j2 文件) | **核心** | Jinja2 模板文件：`chat_system_1.2.j2`、`gen_sql_system_1.2.j2`、`gen_metrics_system_2.0.j2`、`ask_metrics_system_1.0.j2`、`plan_mode_system_2.0.j2`、`evaluation_2.1.j2`、`etl_system_1.1.j2`、`schema_linking_system_1.2.j2`、`compare_system_1.1.j2` 等 |

---

### 3.11 `datus/schemas/` — 数据模式定义（核心，43 文件）

| 分类 | 关键文件 | 用途 |
|---|---|---|
| **基础** | `base.py` | Pydantic 基类 |
| | `agent_models.py` | Agent 配置/状态 Schema |
| | `node_models.py` | 节点输入/输出 Schema |
| | `tool_models.py` | 工具定义 Schema |
| **事件系统** | `action_bus.py` | 动作总线事件 Schema |
| | `action_history.py` | 动作历史记录 |
| | `action_content_builder.py` | 动作内容构建 |
| | `batch_events.py` | 批处理事件 |
| | `interaction_event.py` | 交互事件 |
| | `message_content.py` | 消息内容 Schema |
| **节点专用** (20+) | `gen_sql_agentic_node_models.py` | SQL 生成节点 Schema |
| | `chat_agentic_node_models.py` | 对话节点 Schema |
| | `explore_agentic_node_models.py` | 探索节点 Schema |
| | `ask_metrics_agentic_node_models.py` | 指标问答节点 Schema |
| | `gen_dashboard_agentic_node_models.py` | 仪表板生成节点 Schema |
| | `gen_report_agentic_node_models.py` | 报表生成节点 Schema |
| | `gen_skill_agentic_node_models.py` | 技能生成节点 Schema |
| | `gen_visual_dashboard_models.py` | 可视化仪表板 Schema |
| | `gen_visual_report_models.py` | 可视化报表 Schema |
| | `compare_node_models.py` | 对比节点 Schema |
| | `date_parser_node_models.py` | 日期解析节点 Schema |
| | `doc_search_node_models.py` | 文档搜索节点 Schema |
| | `feedback_agentic_node_models.py` | 反馈节点 Schema |
| | `fix_node_models.py` | 修复节点 Schema |
| | `reason_sql_node_models.py` | SQL 推理节点 Schema |
| | `schema_linking_node_models.py` | Schema 链接节点 Schema |
| | `search_metrics_node_models.py` | 指标搜索节点 Schema |
| | `semantic_agentic_node_models.py` | 语义节点 Schema |
| | `sql_summary_agentic_node_models.py` | SQL 摘要节点 Schema |
| | `subworkflow_node_models.py` | 子工作流节点 Schema |
| | `parallel_node_models.py` | 并行节点 Schema |
| | `scheduler_agentic_node_models.py` | 调度节点 Schema |
| **可视化** | `visualization.py` | 可视化组件 Schema |
| **其他** | `web_result.py` | Web 搜索结果 Schema |
| | `token_usage.py` | Token 用量统计 Schema |
| | `tool_summary.py` | 工具摘要 Schema |
| | `artifact_manifest.py` | 物清单 Schema |
| | `analysis_artifacts.py` | 分析物 Schema |
| | `key_tables_schema.py` | 关键表 Schema |
| | `at_context.py` | @ 上下文 Schema |

---

### 3.12 `datus/utils/` — 共享工具库（核心，38 文件）

| 分类 | 关键文件 | 用途 |
|---|---|---|
| **异常处理** | `exceptions.py` | `DatusException` + 错误码体系（1xxxxx 通用、2xxxxx Node、3xxxxx Model、4xxxxx 工具/存储、5xxxxx 数据库、6xxxxx 语义） |
| **日志** | `loggings.py` | 统一日志接口：`get_logger()`，禁止 `print()` |
| **异步** | `async_utils.py` | 异步工具（事件循环、超时控制） |
| | `async_debug.py` | 异步调试工具 |
| **SQL** | `sql_utils.py` | SQL 解析/格式化/验证 |
| **路径** | `path_manager.py` | 路径管理器 |
| | `path_utils.py` | 路径处理工具 |
| | `reference_paths.py` | 引用路径解析 |
| **消息** | `message_utils.py` | 消息格式转换/处理 |
| | `feedback_prompt.py` | 反馈提示生成 |
| **常量** | `constants.py` | 全局常量定义 |
| **模式** | `schema_utils.py` | Schema 处理工具 |
| **JSON** | `json_utils.py` | JSON 解析/序列化工具 |
| **流式** | `stream_output.py` | 流式输出管理 |
| **子 Agent** | `sub_agent_manager.py` | 子 Agent 生命周期管理 |
| **MCP** | `mcp_decorators.py` | MCP 工具装饰器 |
| **追踪** | `trace_context.py` | 分布式追踪上下文 |
| | `traceable_utils.py` | 可追踪工具 |
| **压缩** | `compress_utils.py` | 数据压缩工具 |
| **文本** | `text_utils.py` | 文本处理工具 |
| **时间** | `time_utils.py` | 时间处理工具 |
| **CSV** | `csv_utils.py` | CSV 读写工具 |
| **Rich** | `rich_util.py` | Rich 库扩展工具 |
| **PyArrow** | `pyarrow_utils.py` | PyArrow 工具 |
| **SSL** | `ssl_utils.py` | SSL 配置工具 |
| **终端** | `terminal_utils.py` | 终端检测/适配 |
| **设备** | `device_utils.py` | 设备信息检测 |
| **环境** | `env.py` | 环境变量管理 |
| **类** | `class_utils.py` | 类加载/反射工具 |
| **资源** | `resource_utils.py` | 资源文件管理 |
| **基准** | `benchmark_utils.py` | 基准测试工具 |
| **多进程** | `multiprocessing_utils.py` | 多进程管理（spawn/fork 策略） |
| **记忆** | `memory_loader.py` | 会话记忆加载 |
| **节点** | `node_utils.py` | 节点通用工具 |
| **工具归档** | `tool_archive.py` | 工具调用归档 |

---

### 3.13 其他辅助模块

| 目录 | 类型 | 用途 | 关键文件说明 |
|---|---|---|---|
| `observability/` | 辅助 | 可观测性（追踪/遥测） | `manager.py`（追踪管理器）、`config.py`（配置）、`registry.py`（注册表）、`reference.py`（引用追踪）、`privacy.py`（隐私过滤）、`native_agents.py`（原生 Agent 埋点）、`openai_agents.py`（OpenAI Agent 埋点）、`adapters/otlp.py`（OpenTelemetry OTLP 导出）、`adapters/langfuse.py`（Langfuse 导出）、`adapters/base.py`（适配器基类）、`adapters/platforms.py`（平台适配） |
| `plugins/` | 辅助 | 插件系统 | `base.py`（插件契约接口）、`registry.py`（插件注册表）、`store.py`（插件存储）、`activation.py`（插件激活/停用）、`runtime_context.py`（运行时上下文）、`prompt.py`（系统提示注入） |
| `auth/` | 辅助 | 认证模块 | `oauth_manager.py`（OAuth2 完整流程）、`oauth_config.py`（OAuth 配置）、`pkce.py`（PKCE 安全流程）、`claude_credential.py`（Claude API 凭证管理）、`token_storage.py`（Token 持久化） |
| `validation/` | **核心** | 输出校验与质量保障 | `builtin_checks.py`（内建校验规则集）、`hook.py`（校验钩子框架）、`llm_runner.py`（LLM-as-Judge 评估）、`report.py`（校验报告生成）、`scheduler_runtime.py`（调度时校验）、`target_extractor.py`（目标提取器） |
| `resources/skills/` | 配置 | 内置技能包（23 个） | `init/`（项目初始化）、`build-kb/`（知识库构建）、`gen-metrics/`（指标生成）、`gen-table/`（表生成）、`semantic-sql-history-profiler/`（SQL 历史画像）、`extract-knowledge/`（知识提取）、`sql-modeling-preflight/`（SQL 建模预检）、`table-validation/`（表校验）、`bi-validation/`（BI 校验）、`transfer-reconciliation/`（数据对账）、`airflow-workflow/`、`superset-dashboard/`、`grafana-dashboard/`、`metricflow-semantic-authoring/`、`osi-metrics-authoring/`、`osi-semantic-authoring/`、`scheduler-validation/`、`create-skill/`、`optimize-skill/`、`data-migration/`、`memory-organization/`、`session-summarize/`、`storage-classify/` |
| `sample_data/` | 配置 | 示例数据 | `california_schools/`（CSV 数据 + SQLite DB + 参考 SQL + 参考模板 + 成功案例 CSV）、`duckdb-demo.duckdb`（DuckDB 演示库）、`superset/`（Helm values + 启停脚本） |
| `conf/` | 配置 | LLM 供应商定义 | `providers.yml`（与根 `conf/providers.yml` 同步） |

---

## 四、`tests/` — 测试套件

### 4.1 `tests/` 根层

| 文件/目录 | 类型 | 用途 | 关键文件说明 |
|---|---|---|---|
| `conftest.py` | **测试** | 共享 Fixture | 全局 pytest fixture（Agent 实例、临时 DB 等） |
| `nightly_requirements.py` | **测试** | 夜间依赖检查 | 检查夜间测试所需 API 密钥是否可用 |
| `__init__.py` | **测试** | 包初始化 | |
| `conf/` | 配置 | 测试配置 | `agent.yml`、`agent_llm_skill.yml`、`wrong_nodes_agent.yml`（错误配置用例） |
| `data/` | 配置 | 测试固件 | 各节点输入 YAML（`GenerateSQLInput.yaml` 等 11 个）、`test_reflection.yaml`、`test_workflow.yaml`、`SSB.db`（Star Schema Benchmark）、`datus_metricflow_db/duck.db`、`semantic_models/bird_school/frpm.yml`、`skills/`（5 个测试技能） |

---

### 4.2 `tests/unit_tests/` — 单元测试（1:1 镜像源码结构）

| 子目录 | 类型 | 测试内容 | 关键测试文件 |
|---|---|---|---|
| `agent/` | **测试** | Agent 核心 | `test_agent.py`（Agent 主流程）、`test_workflow.py`（工作流）、`test_workflow_runner.py`（执行器）、`test_plan.py`（计划）、`test_evaluation.py`（评估） |
| `agent/node/` (60+ 文件) | **测试** | 所有节点 | `test_agentic_node.py`（核心引擎）、`test_gen_sql_agentic_node.py`、`test_chat_agentic_node.py`、`test_compact_*.py`（压缩）、`test_semantic_authoring.py` 等 |
| `api/` | **测试** | API 层 | `test_api_endpoints.py`、`test_service.py`、`auth/`、`models/`、`routes/`、`services/`、`hooks/`、`utils/` |
| `auth/` | **测试** | 认证模块 | OAuth/PKCE/Claude 凭证测试 |
| `build_scripts/` | **测试** | 构建脚本 | `test_build_test_data.py` |
| `ci/` (20 文件) | **测试** | CI 脚本 | `test_audit_tests.py`、`test_run_pr_tests.py`、`test_nightly_manifest.py`、`test_release_workflows.py` 等 |
| `ci/harness/` | **测试** | CI 工具 | `test_check_nightly_coverage.py`、`test_validate_coverage_map.py` |
| `cli/` (80+ 文件) | **测试** | CLI/TUI | `test_autocomplete.py`、`test_chat_commands.py`、`test_bash_mode.py`、`test_service_commands.py`、`action_display/`（6）、`screen/`（2）、`tui/`（17）、`web/`（4） |
| `configuration/` | **测试** | 配置系统 | `test_agent_config.py`、`test_agent_config_loader.py`、`test_project_config.py` |
| `gateway/` | **测试** | IM 网关 | `adapters/`（飞书/Slack）、`channel/`、`richtext/`（富文本 IR） |
| `models/` (17 文件) | **测试** | LLM 适配层 | `test_claude_model.py`、`test_openai_compatible*.py`、`test_session_manager.py`、`test_litellm_adapter.py` 等 |
| `observability/` | **测试** | 可观测性 | 追踪/遥测测试 |
| `plugins/` | **测试** | 插件系统 | 插件注册、激活测试 |
| `prompts/` | **测试** | 提示词 | 提示词模板渲染测试 |
| `schemas/` (19 文件) | **测试** | 数据模式 | Pydantic Schema 校验测试 |
| `storage/` | **测试** | 存储层 | `document/`（含 chunker/cleaner/fetcher/nav_resolver/parser）、`feedback/`、`metric/`、`rdb/`、`reference_sql/`、`reference_template/`、`schema_metadata/`、`semantic_model/`、`table_semantic_profile/`、`task/`、`vector/`，以及 `test_base.py`、`test_backend_holder.py`、`test_embedding_*.py` |
| `tools/` (60+ 文件) | **测试** | 工具层 | 36 func_tool 测试、9 db_tools 测试、9 skill_tools 测试、5 mcp_tools 测试、6 permission 测试等 |
| `utils/` (30+ 文件) | **测试** | 工具库 | 异常/日志/异步/SQL/路径等工具测试 |
| `validation/` (7 文件) | **测试** | 输出校验 | 校验钩子、内建检查、LLM judge 测试 |
| 根层 | **测试** | 顶级模块 | `test_main.py`、`test_mcp_server.py`、`test_multi_round_benchmark.py`、`test_package_version.py`、`test_saas_adaptation.py`、`mock_llm_model.py`（共享 LLM Mock） |

---

### 4.3 `tests/integration/` — 集成测试

| 子目录 | 类型 | 测试内容 | 关键测试文件 |
|---|---|---|---|
| `adapters/` | **测试** | 数据库适配器 | 9 种数据库：`test_postgresql.py`、`test_mysql.py`、`test_clickhouse.py`、`test_doris.py`、`test_greenplum.py`、`test_hive.py`、`test_spark.py`、`test_starrocks.py`、`test_trino.py`；语义适配器：`test_semantic_metricflow_*.py` |
| `agent/` (18 文件) | **测试** | Agent E2E | `test_ask_metrics_agentic.py`、`test_gen_dashboard_agentic.py`、`test_plan_mode_agentic.py`、`test_workflow_orchestration.py` 等 |
| `api/` | **测试** | API E2E | `test_api.py`、`test_api_chat.py`、`test_chat_stream_token_usage.py` |
| `cli/` | **测试** | CLI E2E | `test_cli_commands.py`、`test_cli_textual.py`、`test_interactive_init.py`、`test_quickstart_smoke.py` |
| `gateway/` | **测试** | 网关 E2E | `test_gateway_bridge_agentic.py` |
| `models/` | **测试** | 模型集成 | `test_claude_model.py`、`test_claude_subscription.py`、`test_codex_model.py` 等 |
| `storage/` | **测试** | 存储集成 | `test_doc_search.py`、`test_fastembed_lancedb_smoke.py`、`test_platform_doc*.py`、`test_storage_layout.py` |
| `tools/` (15 文件) | **测试** | 工具集成 | 各工具端到端测试 + `db_tools/test_connector_duckdb.py` |
| 根层 | **测试** | OSI 闭包 | `test_osi_authoring_closure.py` |

---

### 4.4 `tests/regression/` — 回归测试

| 文件 | 类型 | 用途 |
|---|---|---|
| `test_regression_llm.py` | **测试** | LLM 多模型回归（需 API 密钥） |
| `test_regression_web_e2e.py` | **测试** | Web E2E 回归 |
| `conftest.py` | **测试** | 回归测试 Fixture |

---

## 五、`ci/` — CI 流水线

| 关键文件 | 类型 | 用途 |
|---|---|---|
| `run-pr-tests.py` (37KB) | **构建** | **PR CI 主入口**：验收测试 + 受影响单元测试 + Cobertura 覆盖率 + diff 覆盖率 |
| `run-nightly-tests.sh` (64KB) | **构建** | 完整夜间测试驱动（所有标记 `nightly` 的测试） |
| `audit_tests.py` (65KB) | **构建** | 测试套件质量审计（P0 硬失败、P1 警告） |
| `run-merge-queue-tests.py` (10KB) | **构建** | 合并队列集成测试 |
| `run-tests-and-coverage.py` | **构建** | 简单测试+覆盖率运行器 |
| `check_release_readiness.py` (14KB) | **构建** | 发布就绪检查（版本号、变更日志等） |
| `prepare_release.py` (12KB) | **构建** | 发布准备脚本 |
| `prepare_docs_build.py` (9KB) | **构建** | 文档构建准备 |
| `docs_versioning.py` (9KB) | **构建** | 文档版本管理 |
| `nightly_manifest.py` (15KB) | **构建** | 夜间测试任务清单 |
| `classify_nightly_failures.py` (13KB) | **构建** | 夜间失败智能分类 |
| `collect_nightly_trace_summary.py` (18KB) | **构建** | 夜间追踪摘要 |
| `verify_nightly_adapter_sources.py` (9KB) | **构建** | 夜间适配器源校验 |
| `check_flaky_registry.py` (8KB) | **构建** | 不稳定测试追踪 |
| `flaky-registry.yml` | **构建** | 不稳定测试注册表 |
| `provider_coverage_manifest.py` (13KB) | **构建** | Provider 覆盖率清单 |
| `provider-coverage.yml` | **构建** | Provider 覆盖率配置 |
| `acceptance-gate.md` | **文档** | CI 验收门禁说明 |
| `cross-repo-harness.md` | **文档** | 跨仓库 CI 说明 |
| `format-nightly-feishu-report.js` (15KB) | **构建** | 夜间报告 → 飞书通知 |
| `post-audit-comment.js` | **构建** | 审计结果 → PR 评论 |
| `post-coverage-comment.js` | **构建** | 覆盖率 → PR 评论 |
| `preview-comment.py` | **构建** | PR 预览评论 |
| `build_merge_queue_failure_comment.py` | **构建** | 合并队列失败通知 |
| `pytest_manifest_plugin.py` | **构建** | Pytest 清单插件 |
| `pytest_trace_reference_plugin.py` | **构建** | Pytest 追踪引用插件 |
| `harness/check_nightly_coverage.py` (18KB) | **构建** | 夜间覆盖率校验 |
| `harness/coverage-map.yml` (58KB) | **构建** | 覆盖率映射表（源→测试） |
| `harness/validate_coverage_map.py` (23KB) | **构建** | 覆盖率映射校验 |

---

## 六、`docs/` — 文档站点（MkDocs，中英双语）

| 子目录 | 内容 |
|---|---|
| `getting_started/` | 快速开始、上下文数据工程、仪表板 Copilot |
| `cli/` | 20 个 CLI 命令完整文档 |
| `configuration/` | Agent 配置、数据源配置、节点类型、调度器、语义层、SQL 策略 |
| `API/` | Chat API、模型管理 API、知识库 API、部署指南 |
| `knowledge_base/` | 元数据、指标定义、参考 SQL/模板、语义模型、平台文档 |
| `workflow/` | 工作流 API、节点说明、编排指南 |
| `adapters/` | BI 适配器（Superset/Grafana）、数据库适配器、语义适配器、调度器适配器 |
| `gateway/` | 飞书（Lark）、Slack 集成 |
| `integration/` | MCP 协议集成、记忆系统、输出校验 |
| `subagent/` | 全部子 Agent 类型使用文档 |
| `skills/` | 内置技能使用说明 |
| `benchmark/` | 基准测试框架文档 |
| `plugin/` | 插件开发文档 |
| `training/` | 训练相关文档 |
| `develop/` | 开发者指南（可观测性等） |
| `metricflow/` | MetricFlow 相关 |
| `vscode_extension/` | VS Code 扩展文档 |
| `web_chatbot/` | Web 聊天机器人文档 |
| `qa/` | 常见问题 |
| `assets/` | 图片、SVG 图表、`examples-values.yaml` |
| `stylesheets/extra.css` | 站点自定义样式 |
| `javascripts/analytics.js` | 站点分析脚本 |

---

## 七、`benchmark/` — 基准测试框架

| 子目录/文件 | 用途 |
|---|---|
| `scripts/evaluation.py` | 核心评估：对比 Agent 生成的 SQL 与标准答案 |
| `scripts/gen_benchmark.py` | 基准测试生成 |
| `scripts/gen_exec_result.py` | 执行结果生成 |
| `scripts/gen_multi_benchmark.py` | 多轮基准测试 |
| `scripts/llm_recall.py` | LLM Schema 召回率评估 |
| `scripts/schema_recall_bird.py` | BIRD 数据集 Schema 召回 |
| `scripts/schema_recall_spider2.py` | Spider2 数据集 Schema 召回 |
| `scripts/select_answer.py` | 答案选择 |
| `scripts/selection_results_report.py` | 选择结果报告 |
| `scripts/utils.py` | 公共工具 |
| `semantic_layer/success_story.csv` | 成功案例测试数据 |
| `semantic_layer/testing_set.csv` | 语义层测试集 |
| `spider2/` | **空** — `xlang-ai/Spider2` git 子模块占位（需 `git submodule update --init`） |

---

## 八、`build_scripts/` — 构建与容器化

| 文件 | 用途 |
|---|---|
| `build_pypi_package.py` | PyPI 打包全流程：clean → build → install → test → upload |
| `build_test_data.sh` | 构建测试知识库数据 |
| `Dockerfile` | 容器镜像定义 |
| `docker_build.sh` | Docker 构建脚本 |
| `prefetch_model.sh` | 预下载嵌入/重排序模型 |

---

## 九、`scripts/` — 临时开发/调试脚本

| 文件 | 用途 |
|---|---|
| `run_regression.sh` | 回归测试启动脚本 |
| `spark_pi.py` | Spark 连接冒烟测试（计算 π） |
| `debug_subscription_token.py` | 订阅 Token 调试 |
| `test_subscription_models.py` | 订阅模型测试 |

---

## 十、`conf/` — 部署配置样例

| 文件 | 用途 |
|---|---|
| `agent.yml.example` (23KB) | 完整 Agent 配置样例：模型、数据源、工作流、存储、基准、插件 |
| `providers.yml` (9KB) | LLM 供应商定义（模型列表、endpoint、认证方式） |
| `auth_clients.yml.example` | OAuth/SSO 客户端配置样例 |

---

## 十一、`subject/` — 示例语义模型

| 文件 | 用途 |
|---|---|
| `semantic_models/california_schools/frpm.yml` | Free/Reduced Price Meal 数据语义模型 |
| `semantic_models/california_schools/satscores.yml` | SAT 成绩数据语义模型 |
| `semantic_models/california_schools/schools.yml` | 学校信息数据语义模型 |

---

## 十二、ASCII 架构图

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              Datus-Agent Architecture                           │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────┐               │
│  │                      ENTRY POINTS                             │               │
│  │  datus-cli  │  datus-api  │  datus-mcp  │  datus-gateway     │               │
│  │  (REPL/TUI) │  (FastAPI)  │  (MCP Srv)  │  (Feishu/Slack)    │               │
│  └──────┬──────────┬──────────┬──────────┬──────────────────────┘               │
│         │          │          │          │                                       │
│         ▼          ▼          ▼          ▼                                       │
│  ┌──────────────────────────────────────────────────────────────┐               │
│  │                  PRESENTATION LAYER                            │               │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │               │
│  │  │   CLI    │  │   API    │  │ Gateway  │  │   MCP Server │  │               │
│  │  │ (REPL +  │  │ (routes  │  │ (Feishu/ │  │  (tools      │  │               │
│  │  │  TUI)    │  │ services │  │  Slack)  │  │   exposed)   │  │               │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──────┬───────┘  │               │
│  └───────┼─────────────┼─────────────┼───────────────┼──────────┘               │
│          │             │             │               │                           │
│          └─────────────┼─────────────┼───────────────┘                           │
│                        │             │                                           │
│                        ▼             ▼                                           │
│  ┌──────────────────────────────────────────────────────────────┐               │
│  │                    ORCHESTRATION LAYER                         │               │
│  │  ┌──────────────────────────────────────────────────────┐    │               │
│  │  │                  Agent (agent.py)                      │    │               │
│  │  │            Plan → Execute → Reflect Loop               │    │               │
│  │  └──────────────────────────┬───────────────────────────┘    │               │
│  │                              │                                 │               │
│  │  ┌───────────────────────────▼──────────────────────────┐    │               │
│  │  │              Workflow Runner                          │    │               │
│  │  │    ┌────┐  ┌────────┐  ┌───────┐  ┌──────────┐      │    │               │
│  │  │    │Node│→│Agentic │→│GenSQL │→│ExecuteSQL│→ ...   │    │               │
│  │  │    └────┘  │ Node   │  │ Node  │  │   Node   │      │    │               │
│  │  │            └────────┘  └───────┘  └──────────┘      │    │               │
│  │  └──────────────────────────────────────────────────────┘    │               │
│  └──────────────────────────┬───────────────────────────────────┘               │
│                              │                                                   │
│              ┌───────────────┼───────────────┐                                   │
│              ▼               ▼               ▼                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                            │
│  │   PROMPTS    │  │    TOOLS     │  │   STORAGE    │                            │
│  │  (Jinja2)   │  │  (func_tool) │  │  (LanceDB +  │                            │
│  │              │  │              │  │   SQLite)    │                            │
│  │ 60+ templates│  │ 39 function  │  │ 14 KB stores │                            │
│  │              │  │    tools     │  │              │                            │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘                            │
│         │                 │                 │                                    │
│         └─────────────────┼─────────────────┘                                    │
│                           │                                                      │
│                           ▼                                                      │
│  ┌──────────────────────────────────────────────────────────────┐               │
│  │                    INFRASTRUCTURE LAYER                         │               │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │               │
│  │  │  Models  │  │  Config  │  │  Utils   │  │  Validation  │  │               │
│  │  │ (Claude, │  │ (153KB   │  │ (38 mods)│  │  (hooks +    │  │               │
│  │  │  OpenAI, │  │  schema) │  │          │  │   LLM judge) │  │               │
│  │  │  Qwen,..)│  │          │  │          │  │              │  │               │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘  │               │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐   │               │
│  │  │ Schemas  │  │ Plugins  │  │Observab. │  │    Auth    │   │               │
│  │  │(Pydantic)│  │(registry)│  │(OTLP/    │  │  (OAuth2)  │   │               │
│  │  │          │  │          │  │ Langfuse) │  │            │   │               │
│  │  └──────────┘  └──────────┘  └──────────┘  └────────────┘   │               │
│  └──────────────────────────────────────────────────────────────┘               │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────┐               │
│  │                   EXTERNAL INTERFACES                          │               │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌──────────────────┐   │               │
│  │  │ SQLite │  │ DuckDB │  │Postgres│  │ MySQL/ClickHouse │   │               │
│  │  └────────┘  └────────┘  └────────┘  │ StarRocks/Trino..│   │               │
│  │                                       └──────────────────┘   │               │
│  │  ┌────────┐  ┌────────┐  ┌────────┐                          │               │
│  │  │  MCP   │  │  LLM   │  │  Web   │                          │               │
│  │  │Clients │  │  APIs  │  │ Search │                          │               │
│  │  └────────┘  └────────┘  └────────┘                          │               │
│  └──────────────────────────────────────────────────────────────┘               │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────┐               │
│  │                   SUPPORTING SYSTEMS                           │               │
│  │  ci/ (PR/Nightly/Release)  │  tests/ (Unit/Int/Regression)    │               │
│  │  benchmark/ (BIRD/Spider2) │  docs/ (MkDocs bilingual)        │               │
│  │  .github/ (15 workflows)   │  build_scripts/ (pkg/Docker)     │               │
│  └──────────────────────────────────────────────────────────────┘               │
│                                                                                  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## 十三、模块依赖关系总结

```
Entry Points (cli / api / mcp / gateway)
    │
    ▼
Agent (plan → execute → reflect)
    │
    ├── Workflow / WorkflowRunner
    │       │
    │       └── AgenticNode (197KB 核心引擎)
    │               │
    │               ├── Prompts (Jinja2 模板, 60+)
    │               ├── Tools (func_tool ×39, db_tools, mcp_tools, skill_tools)
    │               │       ├── Permission (权限管控)
    │               │       └── SQL Guard (安全守护)
    │               ├── Storage (知识库: LanceDB + SQLite, 14种存储)
    │               └── Schemas (Pydantic 数据模式, 43个)
    │
    ├── Models (多LLM: Claude, OpenAI, DeepSeek, Qwen, Gemini, GLM...)
    │       └── SessionManager (SQLite 会话管理)
    │
    ├── Configuration (agent_config.py 153KB)
    │
    ├── Validation (输出校验: hooks + LLM-as-judge)
    │
    └── Utils (共享工具库, 38个模块)

Observability ← → Plugins ← → Auth (横切关注点，被各层引用)
```

---

## 十四、统计数据一览

| 维度 | 数据 |
|---|---|
| **三大核心文件** | `agentic_node.py` (197KB)、`agent_config.py` (153KB)、`repl.py` (122KB) |
| **核心模块 (12)** | `agent/`、`api/`、`cli/`、`tools/`、`storage/`、`models/`、`prompts/`、`schemas/`、`configuration/`、`gateway/`、`validation/`、`utils/` |
| **辅助模块 (3)** | `observability/`、`plugins/`、`auth/` |
| **内置技能** | 23 个（`resources/skills/`） |
| **提示模板** | 60+ 个 Jinja2 文件 |
| **工具数量** | func_tool 39 个 + db_tools + mcp_tools + skill_tools + llms_tools 等 |
| **节点类型** | 49 个 agentic node |
| **LLM 适配** | Claude、OpenAI、DeepSeek、通义千问、Kimi、GLM、Gemini、MiniMax + OpenRouter |
| **数据库支持** | SQLite、DuckDB、PostgreSQL、MySQL、ClickHouse、Doris、Greenplum、Hive、Spark、StarRocks、Trino + MetricFlow 语义层 |
| **单元测试** | 完全 1:1 镜像源码结构 |
| **集成测试** | 覆盖所有适配器、18 个 Agent E2E 场景 |
| **CI 流水线** | 15 个 GitHub Actions workflows + PR/Nightly/Merge Queue 三条管线 |
| **文档** | MkDocs 中英双语，覆盖 20+ 主题 |
