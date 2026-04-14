# `deerflow` — DeerFlow Agent 框架核心包

基于 LangGraph + LangChain 构建的**超级 Agent 框架**，封装了模型调用、工具编排、沙箱执行、记忆系统、子 Agent 委派等全部能力，所有功能在线程隔离环境中运行。

---

## 目录结构

```
deerflow/
├── client.py              # 嵌入式 Python 客户端（无需 HTTP 服务）
├── agents/                # Agent 核心：图构建 + 中间件链 + 状态管理
│   ├── lead_agent/        # 主 Agent（工厂 + 系统提示词）
│   ├── middlewares/       # 18 层中间件
│   ├── memory/            # 记忆提取、队列、提示
│   ├── checkpointer/     # 状态持久化
│   ├── factory.py         # SDK 级创建入口
│   ├── features.py        # RuntimeFeatures 与中间件排序
│   └── thread_state.py    # ThreadState schema
├── config/                # 配置系统（Pydantic 模型 + 单例加载器）
├── models/                # LLM 模型工厂（多供应商支持）
├── tools/                 # 工具注册与组装
│   └── builtins/          # 内置工具
├── sandbox/               # 沙箱执行系统（本地 / Docker）
│   └── local/             # 本地文件系统沙箱
├── mcp/                   # MCP 协议集成（多 server 工具加载）
├── skills/                # Skills 发现、加载、解析
├── subagents/             # 子 Agent 委派系统
│   └── builtins/          # 内置子 Agent
├── runtime/               # Gateway 嵌入式运行时
│   ├── runs/              # 运行管理器
│   ├── store/             # 持久化存储
│   └── stream_bridge/     # SSE 流桥接
├── community/             # 社区工具（Tavily、Jina、DuckDuckGo 等）
│   ├── tavily/            # Web 搜索 / 抓取
│   ├── jina_ai/           # Jina Reader API
│   ├── firecrawl/         # Firecrawl 爬取
│   ├── exa/               # Exa 搜索
│   ├── ddg_search/        # DuckDuckGo 搜索
│   ├── image_search/      # 图片搜索
│   ├── infoquest/         # InfoQuest 客户端
│   └── aio_sandbox/       # Docker 异步沙箱
├── guardrails/            # 安全护栏（工具调用授权）
├── reflection/            # 动态模块加载（YAML → Python 对象）
├── uploads/               # 文件上传管理
├── tracing/               # 追踪（LangSmith / Langfuse）
└── utils/                 # 通用工具
```

**设计原则**：Harness 层是可独立发布的包，**绝不**依赖 `app.*`（FastAPI Gateway 层）。该边界由 CI 测试 `test_harness_boundary.py` 强制执行。

---

## Agent 系统 (`agents/`)

### 创建入口

| 入口 | 用途 | 特点 |
|------|------|------|
| `make_lead_agent(config)` | 注册在 `langgraph.json`，LangGraph Server 调用 | 依赖 `config.yaml` 全局配置 |
| `create_deerflow_agent(model, tools, ...)` | SDK 级 API，纯参数传入 | 不读配置文件，适合嵌入式使用 |

`make_lead_agent` 核心流程：

1. 解析运行时参数（`model_name`, `thinking_enabled`, `plan_mode` 等）
2. 模型名称解析（请求 → agent 配置 → 全局默认，带 fallback）
3. `create_chat_model()` 创建 LLM 实例
4. `get_available_tools()` 加载全部工具
5. `_build_middlewares()` 构建中间件链
6. `apply_prompt_template()` 生成系统提示词
7. `create_agent()` 构建 LangGraph 图

### ThreadState

在 `AgentState`（含 `messages`）基础上扩展：

| 字段 | 类型 | 用途 |
|------|------|------|
| `sandbox` | `SandboxState` | 当前沙箱 ID |
| `thread_data` | `ThreadDataState` | 线程级目录路径 |
| `title` | `str` | 自动生成的会话标题 |
| `artifacts` | `list[str]` | 产出文件列表（去重合并） |
| `todos` | `list` | 计划模式任务列表 |
| `uploaded_files` | `list[dict]` | 上传文件元数据 |
| `viewed_images` | `dict[str, ViewedImageData]` | 视觉模型处理的图片 |

### 中间件链（18 层）

中间件按严格顺序组装，每层负责一个关切点：

| # | 中间件 | 职责 |
|---|--------|------|
| 1 | **ThreadDataMiddleware** | 创建线程隔离目录 |
| 2 | **UploadsMiddleware** | 注入上传文件信息 |
| 3 | **SandboxMiddleware** | 获取 / 释放沙箱 |
| 4 | **DanglingToolCallMiddleware** | 补全缺失的 ToolMessage（如用户中断） |
| 5 | **LLMErrorHandlingMiddleware** | 标准化 LLM 调用失败 |
| 6 | **GuardrailMiddleware** | 工具调用授权检查（可选） |
| 7 | **SandboxAuditMiddleware** | 沙箱操作安全审计日志 |
| 8 | **ToolErrorHandlingMiddleware** | 工具异常转为 ToolMessage |
| 9 | **SummarizationMiddleware** | 上下文超限时自动摘要（可选） |
| 10 | **TodoMiddleware** | 计划模式任务追踪（可选） |
| 11 | **TokenUsageMiddleware** | Token 使用量记录（可选） |
| 12 | **TitleMiddleware** | 自动生成会话标题 |
| 13 | **MemoryMiddleware** | 对话入队异步记忆更新 |
| 14 | **ViewImageMiddleware** | 注入图片 base64 给视觉模型 |
| 15 | **DeferredToolFilterMiddleware** | 隐藏延迟工具直到搜索激活（可选） |
| 16 | **SubagentLimitMiddleware** | 限制并发子 Agent 数量（可选） |
| 17 | **LoopDetectionMiddleware** | 检测并打断重复工具调用循环 |
| 18 | **ClarificationMiddleware** | 拦截澄清请求，中断流程（**必须最后**） |

排序关键约束：
- ThreadData 必须在 Sandbox 之前（确保 thread_id 可用）
- Uploads 在 ThreadData 之后（需要 thread_id）
- Memory 在 Title 之后（标题先生成）
- Clarification **必须最后**（拦截澄清请求后中断流程）

---

## 配置系统 (`config/`)

约 22 个 Pydantic 模型文件，覆盖系统各方面：

| 文件 | 对应配置段 | 核心类型 |
|------|-----------|----------|
| `app_config.py` | 根配置 | `AppConfig`, `get_app_config()` |
| `model_config.py` | `models[]` | `ModelConfig`（含 thinking/vision 标志） |
| `tool_config.py` | `tools[]` | `ToolConfig`, `ToolGroupConfig` |
| `sandbox_config.py` | `sandbox` | `SandboxConfig`, `VolumeMountConfig` |
| `extensions_config.py` | `extensions_config.json` | `McpServerConfig`, `ExtensionsConfig` |
| `memory_config.py` | `memory` | `MemoryConfig` |
| `summarization_config.py` | `summarization` | `SummarizationConfig`, `ContextSize` |
| `subagents_config.py` | `subagents` | `SubagentsAppConfig` |
| `agents_config.py` | 自定义 agent | `AgentConfig`, `load_agent_config()` |
| `guardrails_config.py` | `guardrails` | `GuardrailsConfig` |
| `checkpointer_config.py` | checkpointer | `CheckpointerConfig`（memory/sqlite/postgres） |
| `tracing_config.py` | tracing | `TracingConfig`（LangSmith/Langfuse） |
| `paths.py` | 路径 | `Paths`, `get_paths()`, `resolve_virtual_path()` |

关键机制：
- `get_app_config()` 有**文件 mtime 自动热加载**
- 值以 `$` 开头的字段解析为环境变量（如 `$OPENAI_API_KEY`）
- `config_version` 版本检查，启动时比对 example 版本并警告

---

## 模型工厂 (`models/`)

`create_chat_model(name, thinking_enabled)` 通过反射系统实例化 LLM。

| 文件 | 作用 |
|------|------|
| `factory.py` | 核心工厂：合并 thinking 设置、附加 tracing 回调 |
| `vllm_provider.py` | vLLM 0.19.0 兼容层（保留 reasoning 字段） |
| `claude_provider.py` | Anthropic 集成 |
| `openai_codex_provider.py` | OpenAI Responses API |
| `patched_openai/deepseek/minimax.py` | 供应商特定补丁 |
| `credential_loader.py` | 凭证加载辅助 |

支持特性：`supports_thinking`、`supports_vision`、`use_responses_api`，以及 vLLM 的 `enable_thinking` 透传。

---

## 工具系统 (`tools/`)

`get_available_tools()` 组装四大来源的工具：

1. **Config 工具**：`config.yaml` 中 `tools[]` 通过 `resolve_variable("module:var")` 动态加载
2. **内置工具**：`present_file`, `ask_clarification`, `view_image`, `task`, `setup_agent`, `tool_search`
3. **MCP 工具**：从 `extensions_config.json` 中启用的 MCP Server 加载，带 mtime 缓存失效
4. **ACP 工具**：`invoke_acp_agent` 调用外部 ACP 兼容 Agent

工具加载时还会根据运行时条件动态增减：
- `view_image` 仅在 `supports_vision=True` 时加载
- `task` 仅在 `subagent_enabled=True` 时加载
- `skill_manage` 仅在 `skill_evolution.enabled=True` 时加载
- `tool_search` 仅在 `tool_search.enabled=True` 时加载

---

## 沙箱系统 (`sandbox/`)

提供安全的代码执行环境。

| 组件 | 作用 |
|------|------|
| `Sandbox`（抽象类） | 定义 `execute_command`, `read_file`, `write_file`, `list_dir` |
| `LocalSandboxProvider` | 本地文件系统，单例模式，带路径映射 |
| `AioSandboxProvider`（community/） | Docker 隔离执行 |
| `SandboxMiddleware` | 管理沙箱生命周期（acquire → use → release） |

### 虚拟路径系统

Agent 看到的路径与物理路径的映射：

| Agent 视角 | 物理路径 |
|-----------|---------|
| `/mnt/user-data/workspace` | `.deer-flow/threads/{tid}/user-data/workspace` |
| `/mnt/user-data/uploads` | `.deer-flow/threads/{tid}/user-data/uploads` |
| `/mnt/user-data/outputs` | `.deer-flow/threads/{tid}/user-data/outputs` |
| `/mnt/skills` | `deer-flow/skills/` |

沙箱工具：`bash`（命令执行 + 路径翻译）、`ls`、`read_file`、`write_file`、`str_replace`、`glob`、`grep`。

---

## 记忆系统 (`agents/memory/`)

长期记忆，跨会话持久化。

### 数据流

1. `MemoryMiddleware` 过滤消息（用户输入 + 最终 AI 回复）→ 入队
2. `queue.py` 去抖（默认 30 秒），按线程去重
3. 后台线程调用 LLM 提取上下文更新和事实
4. 原子写入 `memory.json`（临时文件 + rename）
5. 下次交互时将 top 15 事实 + 上下文注入系统提示词 `<memory>` 标签

### 数据结构（`memory.json`）

- **User Context**：`workContext`, `personalContext`, `topOfMind`
- **History**：`recentMonths`, `earlierContext`, `longTermBackground`
- **Facts**：`id`, `content`, `category`（preference / knowledge / context / behavior / goal）, `confidence`, `createdAt`

---

## 子 Agent 系统 (`subagents/`)

通过 `task` 工具将复杂工作委派给专门 Agent。

- **内置类型**：`general-purpose`（全工具集不含 task）、`bash`（命令专家）
- **执行模型**：双线程池 — `_scheduler_pool`(3) + `_execution_pool`(3)
- **并发限制**：`MAX_CONCURRENT_SUBAGENTS = 3`，由 `SubagentLimitMiddleware` 强制裁切
- **超时**：15 分钟
- **事件流**：`task_started` → `task_running` → `task_completed` / `task_failed` / `task_timed_out`

---

## MCP 集成 (`mcp/`)

连接外部 MCP 服务器以扩展工具能力。

| 文件 | 作用 |
|------|------|
| `cache.py` | 工具缓存 + mtime 失效检测 |
| `client.py` | 构建 server 连接参数（stdio / SSE / HTTP） |
| `tools.py` | `MultiServerMCPClient` 多服务器管理 |
| `oauth.py` | HTTP/SSE MCP 的 OAuth 令牌刷新 |

---

## Skills 系统 (`skills/`)

Agent 可发现和使用的技能。

- **格式**：目录 + `SKILL.md`（YAML 前置元数据：name, description, license, allowed-tools）
- **加载**：`load_skills()` 递归扫描 `skills/{public,custom}/`
- **注入**：启用的技能列表写入系统提示词
- **安装**：`install_skill_from_archive()` 从 .skill ZIP 解压到 custom/
- **安全**：`security_scanner.py` 扫描技能内容

---

## 嵌入式客户端 (`client.py`)

`DeerFlowClient` 是无 HTTP 依赖的进程内客户端，复用与 LangGraph Server / Gateway 相同的模块：

```python
from deerflow.client import DeerFlowClient

client = DeerFlowClient()

# 同步聊天
response = client.chat("Analyze this paper", thread_id="my-thread")

# 流式输出
for event in client.stream("hello"):
    print(event)  # StreamEvent: values / messages-tuple / custom / end
```

还提供 Gateway 等价方法：`list_models()`, `get_mcp_config()`, `list_skills()`, `get_memory()`, `upload_files()`, `get_artifact()` 等，返回格式与 Gateway API 一致。

---

## 其他子系统

| 子系统 | 作用 |
|--------|------|
| `runtime/` | Gateway 嵌入模式：`RunManager` + `run_agent()` + `StreamBridge`（内存队列），SSE 序列化 |
| `guardrails/` | 工具调用前授权检查：`AllowlistProvider`（内置）或自定义 Provider |
| `reflection/` | `resolve_variable("module:var")` 从 YAML 字符串动态加载 Python 对象 |
| `community/` | 可选集成：Tavily、Jina AI、Firecrawl、DuckDuckGo、Exa、ImageSearch、InfoQuest、AioSandbox |
| `tracing/` | LangSmith / Langfuse 追踪回调构建 |
| `uploads/` | 线程级文件上传管理（PDF / PPT / Excel / Word 自动转换） |
| `utils/` | 文件转换、Readability 提取、端口分配 |

---

## 数据流总结

一个用户请求的完整路径：

```
用户消息
  │
  ▼
DeerFlowClient.stream() / LangGraph Server / Gateway run_agent()
  │
  ▼
make_lead_agent(config)
  ├── _resolve_model_name()        → 确定 LLM
  ├── create_chat_model()          → 实例化模型
  ├── get_available_tools()        → 组装工具集
  ├── apply_prompt_template()      → 生成系统提示（含 skills / memory / subagents）
  └── _build_middlewares()         → 构建 18 层中间件
  │
  ▼
create_agent(model, tools, middleware, prompt, state_schema=ThreadState)
  │
  ▼
LangGraph 执行循环
  ├── 中间件前处理（ThreadData → Uploads → Sandbox → ...）
  ├── LLM 调用（带 thinking / vision 支持）
  ├── 工具调用（sandbox bash / MCP / ACP / subagent task）
  ├── 中间件后处理（title → memory → loop detection → ...）
  └── 循环直到完成或中断
  │
  ▼
StreamEvent 输出（values / messages-tuple / custom / end）
```
