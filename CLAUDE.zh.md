# Datus-Agent 项目指令

## 概述

AI 驱动的数据分析 Agent：自然语言 → SQL，多数据库支持，RAG 知识库，MCP 协议。

- **技术栈**：Python 3.12+、OpenAI Agents SDK + LiteLLM、LanceDB、FastAPI、FastMCP、Streamlit
- **包管理器**：uv · **许可证**：Apache-2.0

## 构建与运行

```bash
uv sync                                                # 安装依赖
uv run python ci/run-pr-tests.py upstream/main         # PR CI 流水线：验收测试 + 受影响的单元测试 + 覆盖率
uv run pytest -m nightly tests/                        # 夜间测试（需要 API 密钥）
uv run pytest -m "nightly or regression" tests/        # 全量回归测试
uv run ruff format datus/ tests/ && uv run ruff check --fix datus/ tests/  # 代码格式与检查
bash build_scripts/build_test_data.sh                  # 构建测试知识库
```

## 编码规范

- **ruff**：format + lint，行长度 120，`extend-exclude = mcp/`，isort 分组顺序：标准库 → 第三方 → `datus.*`
- **类型**：全面使用类型提示；数据结构使用 Pydantic
- **日志**：`from datus.utils.loggings import get_logger` — 禁止使用 `print()`
- **错误处理**：通过 `datus.utils.exceptions` 抛出 `DatusException(ErrorCode.XXX, ...)`。错误码区间：1xxxxx 通用，2xxxxx 节点，3xxxxx 模型，4xxxxx 工具/存储，5xxxxx 数据库，6xxxxx 语义
- **仅使用英文**：代码、注释、commit/PR 文本中仅使用英文 — 中文仅限面向中文受众的用户文档

### CLI 界面

所有颜色/符号/辅助函数位于 `datus/cli/cli_styles.py` — 使用 `print_*` 辅助函数（`print_error`、`print_success`、`print_warning`、`print_info`、`print_status`、`print_usage`、`print_empty_set`），不要使用内联 Rich 标记。约束条件：

- 颜色不使用 `bold`；`bold` 仅用于标题/提示标签
- Unicode `✓`/`✗` — 新代码中禁止使用 emoji
- 闭合标签使用缩写形式 `[/]`
- 表格：`header_style=TABLE_HEADER_STYLE`；优先使用 `_render_utils.py` 中的 `build_row_table()`
- 代码渲染：所有 `Syntax()` 使用 `CODE_THEME = "monokai"`
- 交互式选择器：从 `cli_styles` 导入 `CLR_CURSOR` / `CLR_CURRENT`

全屏 TUI 组件参照 `ModelApp`（`model_app.py`）：用 `tui_app.suspend_input()` 包裹 `app.run()`，禁止嵌套 `asyncio.run()`，使用 `DynamicContainer` + `Condition` 守卫，通过 `app.exit(result=Selection(...))` 退出。

### 异步测试

使用 `@pytest.mark.asyncio` 和 `pytest_asyncio.fixture`。事件循环辅助工具（特别是 Windows）：`datus/utils/async_utils.py`。

## 架构

### 存储布局

- **按项目（CWD）**：
  - `./subject/{semantic_models, sql_summaries}/` — 知识库内容，锚定在项目根目录
  - `./.datus/skills/` — 项目级技能，会覆盖 `~/.datus/skills`
  - `./.datus/config.yml` — 项目级覆盖 `target`（provider/model）、`default_datasource`、`project_name`。仅限白名单中的键；由 `/model` 命令写入
- **全局，按项目分片**：
  - `~/.datus/sessions/{project}/{session_id}.db`
  - `~/.datus/data/{project}/datus_db/`（LanceDB、文档存储）
  - `~/.datus/{conf, logs, cache, template, run, benchmark, workspace, skills}` — 共享
- **`project_name`**：通过 `_normalize_project_name`（`agent_config.py`）从 CWD 派生；过长路径会附加 md5 后缀
- **`agent.knowledge_base_home` 已移除** — 知识库始终位于 `{project_root}/subject/`；YAML 字段会被静默忽略

### LLM 配置

两层 provider 模型：

1. **Provider 级**（`agent.providers.<name>`，在 `agent.yml` 中）— 推荐方式。仅包含凭据；可用模型来自 `conf/providers.yml`。`/model` CLI 命令切换时不需编辑 YAML。
2. **自定义/遗留**（`agent.models.<name>`）— 用于不在 `providers.yml` 中的自托管端点。

当前选择持久化在 `./.datus/config.yml`：
```yaml
target: { provider: openai, model: gpt-4.1 }
```
解析优先级：`.datus/config.yml` → `agent.yml` 中的 `agent.target`。

### 扩展点

- **新 Node**：在 `datus/agent/node/` 中新增文件，继承 `Node` 或 `AgenticNode`，在 `datus/configuration/node_type.py` 中注册类型，在 `Node.new_instance()`（`node.py`）中添加工厂映射
- **新 LLM provider（现有接口）**：在 `conf/providers.yml` 和 `datus/conf/providers.yml` 中添加条目；可选添加 `model_specs`。无需 Python 代码
- **新 LLM 模型（新 SDK/认证方式）**：在 `datus/models/` 中新增文件，继承 `LLMBaseModel`（`base.py`），在 `MODEL_TYPE_MAP` 中注册，在 `tests/regression/test_regression_llm.py` 的 `PROVIDER_MODELS` 中添加
- **新 MCP 工具**：在 `datus/tools/func_tool/` 中新增函数，在 MCP server 工具列表中注册

### 权限配置的可变性

- `AgentConfig.config_mutable` 控制权限提示是否可以提供项目范围的持久化选项。
- 当 `config_mutable=False`（多租户 API/网关配置通过 `AgentConfig` 提供）时，Bash 和 SQL 提示仅提供 once/session/deny 选项；不得提供或写入项目授权到 `./.datus/config.yml`。
- 通过 `AgentConfig` 提供的现有 `project_bash_allow` 和 `project_sql_allow` 授权在只读配置模式下仍然有效。
- CLI 流程默认为 `config_mutable=True`，保留项目范围的持久化。

## 防护规则

- **禁止直接数据库导入**：使用 `ConnectorRegistry` / `db_manager_instance`
- **禁止在 Node 中硬编码 LLM 调用**：通过 `LLMBaseModel` 进行
- **禁止 CI 测试中使用外部依赖**：零 API 密钥、零预构建数据、零网络
- **禁止代码中包含密钥**：使用环境变量或 `agent.yml` 中的 `${ENV_VAR}` 替换
- **通过 YAML 配置**：新的可调参数应归属到 `agent.yml`，而非硬编码常量

## PR 规范

### 标题

必须以以下之一开头：`[BugFix]` `[Enhancement]` `[Feature]` `[Refactor]` `[UT]` `[Doc]` `[Tool]` `[Others]`。CI 会拒绝无类型标签的标题。

### 正文 — 必须遵循 `.github/PULL_REQUEST_TEMPLATE.md`

**不可协商。** 每个 PR 正文必须逐字使用模板，并填写全部三个部分：

1. **`## Why`** — 解决的问题；如有相关 issue 请附上链接
2. **`## Solution`** — 方法、关键决策、权衡
3. **`## Test Cases`** — 新增/变更的集成测试或夜间测试；如无，需说明理由

使用 `gh pr create --body` 时，以 `.github/PULL_REQUEST_TEMPLATE.md` 为起点进行复制。缺少或留空任一部分的 PR 须在 review 前修订。

## Commit 工作流

在 push 之前运行与普通 PR 相同的门禁检查，并将额外的全量测试套件针对高风险变更保留。

1. **预格式化**：在暂存之前运行 `uv run ruff format datus/ tests/ && uv run ruff check --fix datus/ tests/`。CI 使用 `ruff format --check datus/ tests/` 和 `ruff check datus/ tests/` 检查相同的路径。
2. **PR 覆盖率流水线**：`uv run python ci/run-pr-tests.py upstream/main`。运行固定的验收测试流水线、从 diff 中选取的受影响单元测试、Cobertura 覆盖率以及 diff 覆盖率。失败时检查 `ci/test-report.md` 和 `ci/diff-cover-report.md`。
3. **测试质量审计**：`uv run python ci/audit_tests.py --repo-root . --diff-only upstream/main` — 必须报告 **`P0=0`**。P0 会导致 CI 硬性失败；P1 仅警告但仍应处理。当修改了大量测试文件时，使用 `--all` 进行全量扫描。仅在合理情况下使用 `# audit-noqa: <rule>` 忽略 noqa。
4. **合并队列演练**：当修改验收流水线目标、CI 脚本或可能影响仅合并队列集成覆盖的代码时，运行 `uv run python ci/run-merge-queue-tests.py`。
5. **Pre-commit hooks**：禁止使用 `--no-verify`；自动修复并重试直到通过。
6. **Push**：仅 push 到 `origin`，禁止 push 到 `upstream`。
7. **PR 正文**：参见上方 **PR 规范 → 正文**。

## 测试规则

### 层级与 Mock 策略

| 层级 | 标记 | Mock 策略 |
|------|------|-------------|
| CI | PR 验收流水线 + 受影响的 `tests/unit_tests/`；<5 秒/测试，确定性 | **必须** mock 所有外部调用（LLM、远程数据库、网络、可选包） |
| Nightly | `@pytest.mark.nightly` | 可使用真实 LLM API；mock 不稳定的服务 |
| Regression | `@pytest.mark.regression` | 真实服务；缺少密钥时使用 `@pytest.mark.skipif` 跳过 |

CI 在缺少可选包（`datus-bi-superset`、`datus-bi-grafana` 等）的环境下运行。涉及导入这些包的代码的测试，无论包是否已安装都必须能正常工作。（`datus-bi-core` 是硬依赖，始终可用。）

### 文件命名与位置

| 位置 | 模式 |
|----------|---------|
| `tests/unit_tests/` | `test_{module}.py`，**镜像**源码路径：`datus/a/b/c.py` → `tests/unit_tests/a/b/test_c.py`（例如 `datus/utils/json_utils.py` → `tests/unit_tests/utils/test_json_utils.py`）|
| `tests/integration/` | `test_{scenario}.py` |
| `tests/regression/` | `test_regression_{dimension}.py` |

新增子目录时创建中间 `__init__.py`。常用模式：`@pytest.mark.skipif(not os.getenv("KEY"), reason=...)` 用于缺少 API 密钥；`@pytest.mark.parametrize("db_type", [DBType.SQLITE, DBType.DUCKDB])` 用于跨数据库参数化测试。

### 修改以下模块时必须添加测试

单元测试遵循上述映射规则。下表列出了除映射规则外需要的**额外**集成/回归测试：

| 修改的模块 | 额外测试 |
|---|---|
| `datus/models/{provider}_model.py` | `integration/models/test_*_model.py`、`regression/test_regression_llm.py` |
| `datus/agent/node/` | `unit_tests/agent/node/test_node.py`、`test_schema_linking.py`、`test_date_parser_*.py` |
| `datus/cli/repl.py` | `integration/cli/test_cli_commands.py`、`regression/test_regression_web_e2e.py` |
| `datus/tools/func_tool/` | `integration/tools/test_func_tools_db.py`、`integration/tools/test_mcp_server.py` |
| `datus/tools/skill_tools/` | `unit_tests/tools/skill_tools/test_skill_*.py` |
| `datus/tools/permission/` | `unit_tests/tools/permission/test_permission_*.py` |
| `datus/mcp_server.py` | `unit_tests/test_mcp_server.py`、`integration/tools/test_mcp_server.py` |
| `datus/storage/reference_template/` | `unit_tests/storage/reference_template/test_*.py`、`integration/tools/test_reference_template.py` |
| `datus/storage/document/` | `integration/storage/test_doc_search.py`、`integration/storage/test_platform_doc.py` |

### 测试质量（超越覆盖率）

除 happy path 外，还需覆盖：**输入格式变体**（不仅仅是常见格式，而是所有有效形式）；**返回类型契约**（每个分支返回相同结构）；**跨组件契约**（使用生产者的真实输出进行消费）；对 regex/SQL/path 沙箱进行**对抗性输入**测试；**递归/嵌套结构**深度 ≥ 3；对标准的**规范合规性**（`.gitignore`、SQL 方言）。

### 删除代码时禁止添加墓碑测试

删除代码时，应更新或删除覆盖该代码的现有测试。**禁止**添加断言代码/函数/字符串不再存在的测试（`not hasattr(...)`、`"xxx" not in source`、导入失败检查等）— 这类测试只是固定了实现细节，永远抓不到真正的 bug，并会阻碍未来的合理重构。删除是否安全的证明是现有行为测试通过，而非新增的否定断言。

例外：当删除本身构成外部契约时，添加回归测试断言该**行为**不再发生，并在测试中注明其保护的契约。适用场景：已弃用的公共 API/配置字段（例如已删除的 YAML 键必须被静默忽略）、安全修复（例如密钥不得出现在日志中）、有真实回归风险的已删除副作用（在 mock 上使用 `assert_not_called`）。
