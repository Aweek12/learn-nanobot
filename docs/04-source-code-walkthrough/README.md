# 04 - 源码逐行解读

本章保留原教程的目录总览与讲解主题，按本地 nanobot `0.4.1`（`27f1a836b5c04a30763dd91960059f81c05bf505`）核对实现。每节保留原讲解主题；方法逻辑变化时，展示本地新版完整方法并添加中文 # 注释，保留签名、装饰器、全部分支及异常处理。长方法可以展开阅读，源码链接用于核对原文件。

## 目录

- [4.1 源码目录结构总览](#section-4-1)
- [4.2 AgentLoop - loop.py](#section-4-2)
- [4.3 AgentRunner - runner.py](#section-4-3)
- [4.4 ContextBuilder - context.py](#section-4-4)
- [4.5 MemoryStore - memory.py](#section-4-5)
- [4.6 SubagentManager - subagent.py](#section-4-6)
- [4.7 工具系统 - tools/](#section-4-7)
- [4.8 MessageBus - bus/](#section-4-8)
- [4.9 Channel 适配层 - channels/](#section-4-9)
- [4.10 配置系统 - config/schema.py](#section-4-10)
- [4.11 关键代码片段解读](#section-4-11)
- [4.12 面试高频题](#section-4-12)
- [4.13 本章总结](#section-4-13)

<a id="section-4-1"></a>

## 4.1 源码目录结构总览

```
nanobot/
├── __init__.py
├── __main__.py                 # 入口点 → 启动 CLI
├── cli.py                      # 命令行接口
│
├── agent/                      # 核心 Agent 模块
│   ├── __init__.py
│   ├── loop.py                 # ★ AgentLoop - 消息消费和会话管理
│   ├── runner.py               # ★ AgentRunner - ReAct 循环执行器
│   ├── context.py              # ★ ContextBuilder - Prompt 构建
│   ├── memory.py               # ★ MemoryStore - 记忆系统
│   ├── subagent.py             # SubagentManager - 子 Agent 管理
│   └── tools/                  # 工具子模块
│       ├── __init__.py
│       ├── registry.py         # ★ ToolRegistry - 工具注册表
│       ├── mcp.py              # ★ MCPToolWrapper - MCP 工具包装器
│       ├── message.py          # MessageTool - 消息回复工具
│       └── spawn.py            # SpawnTool - 子 Agent 生成工具
│
├── bus/                        # 消息总线
│   ├── __init__.py
│   └── message_bus.py          # ★ MessageBus - 双队列消息总线
│
├── channels/                   # 通道适配器
│   ├── __init__.py
│   ├── base.py                 # BaseChannel - 通道基类
│   ├── telegram.py             # Telegram 适配器
│   ├── discord.py              # Discord 适配器
│   ├── feishu.py               # 飞书适配器
│   ├── dingtalk.py             # 钉钉适配器
│   ├── wechat.py               # 微信适配器
│   ├── qq.py                   # QQ 适配器
│   ├── slack.py                # Slack 适配器
│   └── web.py                  # Web 适配器
│
├── providers/                  # LLM 供应商
│   ├── __init__.py
│   ├── base.py                 # BaseProvider - 供应商基类
│   ├── openai.py               # OpenAI 适配
│   ├── anthropic.py            # Anthropic/Claude 适配
│   ├── deepseek.py             # DeepSeek 适配
│   ├── dashscope.py            # 通义千问适配
│   ├── ollama.py               # Ollama 本地模型
│   ├── groq.py                 # Groq 适配
│   └── registry.py             # PROVIDERS 注册表
│
├── config/                     # 配置管理
│   ├── __init__.py
│   ├── schema.py               # 配置数据类定义
│   └── loader.py               # YAML 配置加载
│
└── utils/                      # 工具函数
    ├── __init__.py
    ├── logging.py              # 日志
    └── helpers.py              # 通用辅助函数
```

> 标注 ★ 的文件是核心文件，面试必须掌握。

---


### 新版路径对照（不改动上面的原目录图）

上图保留原教程的导航结构。按本地新版打开文件时，使用下面的对应关系；目录名变化不影响后面按“消息消费、模型循环、上下文、记忆、工具”学习的顺序。

| 原图中的位置 | 本地新版位置或变化 |
| --- | --- |
| `cli.py` | `nanobot/cli/`；启动编排见 `gateway_runtime.py` |
| `bus/message_bus.py` | `nanobot/bus/queue.py`；消息结构在 `bus/events.py` |
| `tools/spawn.py` | `nanobot/agent/tools/subagent.py`；工具名为 `subagent` |
| `channels/telegram.py`、`feishu.py` | `channels/telegram/runtime.py`、`channels/feishu/runtime.py` |
| `providers/base.py` 的 BaseProvider | 现为 `LLMProvider`；结合 `routing.py`、`factory.py` 阅读 |
| 每个服务对应一个 provider 文件 | 多个兼容服务可以共用 `openai_compat_provider.py`，按路由找到实际适配器 |
| `loader.py` 的 YAML 配置 | 当前加载 `config.json`，用 Pydantic 校验 |
| 原图未展开的持久化、安全模块 | 补充阅读 `session/state.py`、`session/sqlite_store.py`、`security/workspace_access.py` |

<a id="section-4-2"></a>

## 4.2 AgentLoop - loop.py

### 文件定位

AgentLoop 负责接收消息、组织会话和安排一轮对话。先问自己：用户消息从哪里来，同一会话的两条消息会不会同时修改历史？这两个问题决定了为什么需要消费循环与会话队列。

源码：[nanobot/agent/loop.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/loop.py#L207)，`AgentLoop`。

### 核心类：AgentLoop

原教程用“每收到一条消息创建一个 Runner”解释 Loop，新版已经不是这个关系。下面是新版完整构造方法，可重点查找 `self.runner = AgentRunner()`：

<details>
<summary>展开 AgentLoop.__init__() 完整方法（187 行源码）</summary>

<a id="method-agentloop-init"></a>

源码：[nanobot/agent/loop.py:274](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/loop.py#L274)，`AgentLoop.__init__` 完整方法。

```python
# 组装 Loop 的执行依赖、会话协调、工具和运行状态；Runner 在这里创建并复用。
def __init__(
    self,
    bus: MessageBus,
    provider: LLMProvider,
    workspace: Path,
    model: str | None = None,
    max_iterations: int | None = None,
    max_concurrent_subagents: int | None = None,
    context_window_tokens: int | None = None,
    max_tool_result_chars: int | None = None,
    provider_retry_mode: str = "standard",
    cron_service: CronService | None = None,
    restrict_to_workspace: bool = False,
    session_manager: SessionManager | None = None,
    tool_registry: ToolRegistry | None = None,
    channels_config: ChannelsConfig | None = None,
    timezone: str | None = None,
    session_ttl_minutes: int = 0,
    hooks: list[AgentHook] | None = None,
    hook_factories: list[AgentTurnHookFactory] | None = None,
    unified_session: bool = False,
    disabled_skills: list[str] | None = None,
    tools_config: ToolsConfig | None = None,
    network_config: NetworkConfig | None = None,
    provider_snapshot_loader: Callable[..., ProviderSnapshot] | None = None,
    provider_signature: tuple[object, ...] | None = None,
    model_presets: dict[str, ModelPresetConfig] | None = None,
    preset_catalog_loader: preset_helpers.PresetCatalogLoader | None = None,
    model_preset: str | None = None,
    dream_model_preset: str | None = None,
    preset_snapshot_loader: preset_helpers.PresetSnapshotLoader | None = None,
    turn_delivery_factory: TurnDeliveryFactory | None = None,
    runtime_model_publisher: Callable[[str, str | None], None] | None = None,
    restart_mode: str = "auto",
    local_trigger_store: LocalTriggerStore | None = None,
    idle_compact_check_interval_seconds: int = 0,
    recovery_admission: RecoveryAdmission | None = None,
    builtin_assembly: Callable[[SessionManager, MessageBus, ToolRegistry, CommandRouter], ExecutionPolicy] | None = None,
):
    from nanobot.config.schema import ToolsConfig

    _tc = tools_config or ToolsConfig()
    defaults = AgentDefaults()
    self.bus = bus
    self.execution_policy: ExecutionPolicy | None = None
    self._recovery_admission = recovery_admission
    if turn_delivery_factory is not None:
        if turn_delivery_factory.bus is not bus:
            raise ValueError("turn delivery factory must use the agent message bus")
        self.turn_delivery_factory = turn_delivery_factory
    else:
        self.turn_delivery_factory = TurnDeliveryFactory(bus)
    self.runtime_event_publisher = self.turn_delivery_factory.runtime_event_publisher
    self.channels_config = channels_config
    self.restart_mode = restart_mode
    self._runtime_model_publisher = runtime_model_publisher
    self.workspace = workspace
    initial_model = model or provider.get_default_model()
    self.max_iterations = (
        max_iterations if max_iterations is not None else defaults.max_tool_iterations
    )
    initial_context_window = (
        context_window_tokens
        if context_window_tokens is not None
        else defaults.context_window_tokens
    )
    configured_presets = model_presets or {}
    self.runtime_resolver = ModelRuntimeResolver(
        LLMRuntime.capture(
            provider,
            initial_model,
            context_window_tokens=initial_context_window,
            snapshot_signature=provider_signature,
        ),
        model_presets=configured_presets,
        preset_catalog_loader=preset_catalog_loader,
        configured_default_preset=model_preset,
        provider_snapshot_loader=provider_snapshot_loader,
        preset_snapshot_loader=preset_snapshot_loader,
    )
    self.dream_model_preset = dream_model_preset
    self.max_tool_result_chars = (
        max_tool_result_chars
        if max_tool_result_chars is not None
        else defaults.max_tool_result_chars
    )
    self.provider_retry_mode = provider_retry_mode
    self.tools_config = _tc
    self.network_config = network_config or NetworkConfig()
    self.web_config = _tc.web
    self.exec_config = _tc.exec
    self.cron_service = cron_service
    self.local_trigger_store = local_trigger_store
    self.restrict_to_workspace = restrict_to_workspace
    self.workspace_scopes = WorkspaceScopeResolver(
        default_workspace=workspace,
        default_restrict_to_workspace=restrict_to_workspace,
    )
    self._start_time = time.time()
    self._extra_hooks: list[AgentHook] = hooks or []
    self._hook_factories: list[AgentTurnHookFactory] = hook_factories or []

    self.context = ContextBuilder(workspace, timezone=timezone, disabled_skills=disabled_skills)
    self.sessions = session_manager or SessionManager(workspace)
    # One file-read/write tracker per logical session. The tool registry is
    # shared by this loop, so tools resolve the active state via contextvars.
    self._file_state_store = FileStateStore(max_sessions=SESSION_CACHE_MAX_SIZE)
    # SessionManager owns every durable deletion entrypoint, including the
    # WebUI and fork rollback paths.  Observe that boundary once instead of
    # duplicating cleanup in each consumer.
    self.sessions.set_delete_observer(self._file_state_store.discard)
    self.tools = tool_registry if tool_registry is not None else ToolRegistry()
    self._exec_session_manager = ExecSessionManager()
    # 在构造阶段创建并复用 Runner，每次 run 的消息与状态仍是独立的。
    self.runner = AgentRunner()
    self.consolidator = Consolidator(
        store=self.context.memory,
        sessions=self.sessions,
        build_messages=self.context.build_messages,
        get_tool_definitions=self.tools.get_definitions,
        resolve_prompt_context=PersistedPromptContextResolver(
            workspace_scopes=self.workspace_scopes,
            unified_session=unified_session,
        ),
    )
    self.subagents = SubagentManager(
        workspace=workspace,
        bus=bus,
        tools_config=_tc,
        network_config=self.network_config,
        max_tool_result_chars=self.max_tool_result_chars,
        restrict_to_workspace=restrict_to_workspace,
        disabled_skills=disabled_skills,
        max_iterations=self.max_iterations,
        max_concurrent_subagents=max_concurrent_subagents,
        consolidator=self.consolidator,
        session_manager=self.sessions,
    )
    self._unified_session = unified_session
    self._running = False
    self._runtime_context_providers: list[RuntimeContextProvider] = []
    self._active_tasks: dict[str, set[asyncio.Task[Any]]] = {}
    self._discarding_sessions: set[str] = set()
    self._background_tasks: set[asyncio.Task[Any]] = set()
    self._close_lock = asyncio.Lock()
    self._session_locks: weakref.WeakValueDictionary[str, asyncio.Lock] = (
        weakref.WeakValueDictionary()
    )
    # A queue is installed before its single session worker is scheduled.
    # It remains the only admission path until that worker atomically
    # observes the queue empty and removes it.
    self._pending_queues: dict[str, asyncio.Queue[InboundMessage]] = {}
    self._preserve_inflight_turns_on_shutdown = False
    self._deferred_automation_turns: dict[str, list[InboundMessage]] = {}
    self._cron_turns = CronTurnCoordinator(
        enqueue=self._enqueue_session_message,
        deferred_queues=self._deferred_automation_turns,
    )
    self._local_trigger_turns = LocalTriggerTurnCoordinator(
        enqueue=self._enqueue_session_message,
        deferred_queues=self._deferred_automation_turns,
    )
    self._automation_turn_coordinators = (
        self._cron_turns,
        self._local_trigger_turns,
    )
    # NANOBOT_MAX_CONCURRENT_REQUESTS: unset or <=0 means unlimited.
    _max = int(os.environ.get("NANOBOT_MAX_CONCURRENT_REQUESTS", "0"))
    self._concurrency_gate: asyncio.Semaphore | None = (
        asyncio.Semaphore(_max) if _max > 0 else None
    )
    self.auto_compact = AutoCompact(
        sessions=self.sessions,
        consolidator=self.consolidator,
        session_ttl_minutes=session_ttl_minutes,
        bind_events=self._idle_events,
        session_lock=self._get_session_lock,
    )
    self._idle_compact_check_interval_s = idle_compact_check_interval_seconds
    self._next_idle_compact_check_at = time.monotonic()
    self._idle_compact_check_running = False
    if model_preset:
        self.set_model_preset(model_preset, publish_update=False)
    self.commands = CommandRouter()
    register_core_commands(self.commands)
    if builtin_assembly is not None:
        self.execution_policy = builtin_assembly(self.sessions, self.bus, self.tools, self.commands)
    self._register_default_tools(provider_snapshot_loader=provider_snapshot_loader)
```

</details>

**关键点解读：**

1. 这行位于 `AgentLoop.__init__`，说明 Runner 对象由 Loop 初始化并复用，不是在消息消费循环中反复创建。
2. 一次 `runner.run(spec)` 才代表一次执行。对象数与执行次数是两个概念；复用对象不等于所有会话共用一份局部消息列表。
3. Loop 管理会话队列和交付；Runner 管理模型与工具的迭代。这样 WebUI、TUI 和聊天渠道可以共用后端执行逻辑。

### 构造函数参数详解

新版构造函数很长，第一遍先看下面几组依赖，其他选项在遇到对应功能时再回头查。

| 依赖或配置 | 要理解的作用 | 与旧示例的差别 |
| --- | --- | --- |
| `bus` | 从统一消息入口接收输入 | 消费调用封装为 `consume_inbound()` |
| `provider` | 提供模型调用能力 | 共享接口为 `LLMProvider` |
| `workspace` | 提供上下文及文件操作的根位置 | 工作目录与安全授权要分别理解 |
| `max_iterations` | 控制模型工具循环预算 | 当前配置 `maxToolIterations` 默认 200，不能沿用旧 40 |
| 会话锁、待处理队列 | 协调同一会话的输入和取消 | 不能简化成每条消息任意启动任务 |
| 全局并发闸门 | 可选的总体并发限制 | 由 `NANOBOT_MAX_CONCURRENT_REQUESTS` 控制，不能写死为 3 |

**理解方法：**先区分“传进来的服务对象”和“控制执行的数值配置”。不要为了记住参数列表而忽略谁负责状态。

### run() 方法：核心消费循环

原教程这个小节仍然讲 `async def run(self)`。新版增加超时维护、取消判断、输入接纳和关闭处理，下面展示完整 `async def run(self) -> None:`，不再只取等待和分发两行。

<a id="method-agentloop-run"></a>

源码：[nanobot/agent/loop.py:1408](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/loop.py#L1408)，`AgentLoop.run` 完整方法。

```python
# 持续消费输入并分发；区分超时、取消和输入拒绝，退出时关闭运行时。
async def run(self) -> None:
    """Run the agent loop, dispatching messages as tasks to stay responsive to /stop."""
    # 标记消费循环已启动，后续 while 根据该状态继续等待输入。
    self._running = True
    # 包含 finally 清理路径，保证正常结束和异常退出都按约定收尾。
    try:
        logger.info("Agent loop started")

        # 循环只负责持续接收与分发，完整模型执行由下层协调。
        while self._running:
            try:
                # Keep dequeue and admission in this task. wait_for would
                # dequeue in another task, leaving a false-idle handoff gap.
                # 只限制本次取消息的等待时间，不限制模型回答时间。
                async with asyncio.timeout(1.0):
                    # 无输入时异步等待，取到输入后进入分发流程。
                    msg = await self.bus.consume_inbound()
            # 队列等待超时走维护检查路径，然后继续消费输入。
            except asyncio.TimeoutError:
                # Storage may wait behind a turn that /stop needs to cancel.
                # 没有新输入时仍按时间条件检查是否需要后台整理会话。
                if self._idle_compact_check_due(time.monotonic()):
                    # 交给后台任务处理，让主循环继续接收输入和停止命令。
                    self.schedule_background(self._check_expired_sessions_if_due())
                continue
            # 单独处理取消信号，避免把真实取消误当成普通运行错误。
            except asyncio.CancelledError:
                # Preserve real task cancellation so shutdown can complete cleanly.
                # Only ignore non-task CancelledError signals that may leak from integrations.
                # 真实关闭或任务取消必须继续传播，不能误当作普通集成错误吞掉。
                if not self._running or task_is_cancelling():
                    raise
                logger.warning(
                    "Ignoring leaked CancelledError while consuming inbound messages"
                )
                continue
            # 其他取消息错误只记录告警并继续循环，避免一次集成错误停止所有输入接收。
            except Exception as e:
                logger.warning("Error consuming inbound message: {}, continuing...", e)
                continue

            # 在输入接纳活动范围内分发，协调运行时忙闲与关闭状态。
            with self.bus.admission.activity():
                try:
                    # 由分发器处理优先命令、会话路由和已有会话队列。
                    await self._dispatch_inbound(msg)
                # 会话队列已满、会话状态冲突或执行控制拒绝时，向用户反馈输入未被接纳。
                except (SessionOperationQueueFullError, SessionConflictError, ExecutionControlError) as exc:
                    key = self._effective_session_key(msg)
                    logger.warning("Session input rejected for {}: {}", key, exc)
                    if msg.is_user_input:
                        delivery = self.turn_delivery_factory.unrouted(msg, key)
                        # 输入被拒绝时反馈失败；是否发布完成事件取决于该会话是否仍有活动队列。
                        await delivery.fail(publish_completion=key not in self._pending_queues)
    # 无论正常返回、异常或取消，都进入这里完成资源与状态收尾。
    finally:
        # 统一关闭 Loop 持有的运行资源，保证异常退出也能清理。
        await self.aclose()
```

完整的 `AgentLoop.run()` 已在[本章前文](#method-agentloop-run)列出，可结合此处讲解对照阅读。

**关键点解读：**

1. `await` 等待队列时让出事件循环，不会让整个进程停止接收其他异步事件。
2. 1 秒是本次取消息的等待窗口，不是模型回答的时间限制，也不是每秒只能处理一条消息。
3. `_dispatch_inbound()` 识别优先命令并路由到会话队列。因此 `/stop` 能走控制路径；不能直接把分发理解为等待一整轮模型回答结束。
4. 同一会话已有活动队列时，新输入进入对应的协调路径；不同会话可以独立推进，是否再受全局闸门限制取决于配置。
5. `finally` 保证退出时清理资源。真实取消要传播出去，这与普通取消息失败后继续等待不同。

**跟一次输入：**用户 A 发“读 README”后，模型可能还在调用工具；此时 A 发 `/stop`，主消费循环仍有机会取到控制消息。若把完整 ReAct 循环直接堵在消费循环中，这种响应性就难以保证。

### _register_default_tools() 方法

保留原教程“默认工具如何注册”的问题，但新版应转到 `ToolLoader.load()`，不要再寻找旧示例中的同名方法。

<a id="method-toolloader-load"></a>

源码：[nanobot/agent/tools/loader.py:89](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/loader.py#L89)，`ToolLoader.load` 完整方法。

```python
# 发现和筛选工具，创建实例，处理名称冲突后注册到工具表。
def load(self, ctx: ToolContext, registry: ToolRegistry, *, scope: str = "core") -> list[str]:
    """Load defaults while preserving tools already supplied by the caller."""
    registered: list[str] = []
    supplied_names = set(registry.tool_names)
    builtin_names: set[str] = set()
    sources = [(self.discover(), False), (self._discover_plugins().values(), True)]
    for source, is_plugin_source in sources:
        for tool_cls in source:
            cls_label = tool_cls.__name__
            try:
                if scope not in getattr(tool_cls, "_scopes", {"core"}):
                    continue
                if not tool_cls.enabled(ctx):
                    continue
                tool = tool_cls.create(ctx)
                if is_plugin_source:
                    tool = _LegacyErrorPrefixTool(tool)
                if tool.name in supplied_names:
                    if not is_plugin_source:
                        builtin_names.add(tool.name)
                    continue
                if registry.has(tool.name):
                    if is_plugin_source and tool.name in builtin_names:
                        logger.warning(
                            "Plugin %s skipped: conflicts with built-in tool %s",
                            cls_label, tool.name,
                        )
                        continue
                    logger.warning(
                        "Tool name collision: %s from %s overwrites existing",
                        tool.name, cls_label,
                    )
                registry.register(tool)
                registered.append(tool.name)
                if not is_plugin_source:
                    builtin_names.add(tool.name)
            # 处理 Exception，按本方法约定记录、转换或继续传播错误。
            except Exception:
                logger.exception("Failed to register tool: %s", cls_label)
        if not is_plugin_source and "rg" in builtin_names:
            replaced = (builtin_names & {"grep", "find_files"}) - supplied_names
            for name in replaced:
                registry.unregister(name)
            registered = [name for name in registered if name not in replaced]
            builtin_names.difference_update(replaced)
    return registered
```

**关键点解读：**发现解决“有哪些工具类”，创建解决“依赖怎么传进去”，注册解决“用什么名字找到实例”。拆开这三步，才能理解为什么新增工具通常无需在 Loop 中继续堆注册语句。具体可用工具还取决于配置，目录中存在文件不等于本轮一定启用。

### _save_turn() 方法

原示例把用户前 200 字和回复前 500 字存到 `HISTORY.md`，这不再是当前方法的实现。新版这里处理的是本轮结构化会话增量，而且方法是同步 `def`。

<details>
<summary>展开 AgentLoop._save_turn() 完整方法（110 行源码）</summary>

<a id="method-agentloop-save-turn"></a>

源码：[nanobot/agent/loop.py:2464](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/loop.py#L2464)，`AgentLoop._save_turn` 完整方法。

```python
# 把本轮新增结构化消息写入会话，校验工具调用配对并处理摘要检查点。
def _save_turn(
    self,
    session: Session,
    messages: list[dict[str, Any]],
    skip: int,
    *,
    turn_latency_ms: int | None = None,
    summary_checkpoint: SessionSummaryCheckpoint | None = None,
    input_persisted_early: bool = False,
) -> None:
    declared_tool_call_ids = {
        str(tc["id"])
        for m in session.messages
        if m.get("role") == "assistant"
        for tc_value in cast(Iterable[object], m.get("tool_calls") or [])
        if isinstance(tc_value, dict)
        for tc in (cast(dict[str, Any], tc_value),)
        if tc.get("id")
    }
    fulfilled_tool_call_ids = {
        str(m["tool_call_id"])
        for m in session.messages
        if m.get("role") == "tool" and m.get("tool_call_id")
    }
    last_assistant_idx: int | None = None
    saved_followup_ids: set[str] = set()
    # 校验摘要边界，避免消息过滤后把摘要插入错误位置。
    checkpoint_boundary = self._validated_checkpoint_boundary(
        summary_checkpoint,
        skip=skip,
        message_count=len(messages),
        session_key=session.key,
    )

    # The trigger input may already be the session tail while still being
    # the first message after the replacement checkpoint.
    if summary_checkpoint is not None and checkpoint_boundary == skip - 1:
        insert_at = len(session.messages) - (1 if input_persisted_early else 0)
        # 在原始 transcript 对应边界插入摘要检查点。
        session.commit_summary_checkpoint(
            summary_checkpoint.summary,
            insert_at=insert_at,
        )

    # 从新增消息下标开始保存，skip 不是截断每条消息的字符长度。
    for message_index, message in enumerate(messages[skip:], start=skip):
        # Insert against the raw transcript index before filtering the
        # message so persistence cleanup cannot shift the H/Δ boundary.
        if summary_checkpoint is not None and checkpoint_boundary == message_index:
            # 在原始 transcript 对应边界插入摘要检查点。
            session.commit_summary_checkpoint(summary_checkpoint.summary)

        # 把模型消息转换成适合会话持久化的条目。
        entry = persisted_message(message)
        followup_id = entry.pop(PENDING_FOLLOWUP_ID_KEY, None)
        # 根据角色区分用户、assistant 和工具结果的保存约束。
        role, content = entry.get("role"), entry.get("content")
        # 空 assistant 且没有工具调用时跳过，防止污染后续请求。
        if role == "assistant" and not content and not entry.get("tool_calls"):
            continue  # skip empty assistant messages — they poison session context
        # 工具结果需要匹配已声明且尚未完成的调用 ID，防止重放时破坏消息结构。
        if role == "tool":
            tool_call_id = entry.get("tool_call_id")
            tool_call_id_str = str(tool_call_id) if tool_call_id else ""
            if (
                not tool_call_id_str
                or tool_call_id_str not in declared_tool_call_ids
                or tool_call_id_str in fulfilled_tool_call_ids
            ):
                # Undeclared tool results corrupt future provider requests.
                logger.warning(
                    "Dropping invalid tool result {} from session {} during persistence",
                    tool_call_id_str or "(missing id)",
                    session.key,
                )
                continue
            fulfilled_tool_call_ids.add(tool_call_id_str)
            # Preserve model-visible text for replay; redact only inline images.
            if isinstance(content, list):
                filtered = self._sanitize_persisted_blocks(
                    cast(list[object], content),
                )
                if not filtered:
                    # Preserve the tool_call/result pair after block filtering.
                    filtered = [
                        {"type": "text", "text": "[tool result omitted during persistence]"}
                    ]
                entry["content"] = filtered
        elif role == "user":
            if isinstance(content, list):
                filtered = self._sanitize_persisted_blocks(
                    cast(list[object], content),
                )
                if not filtered:
                    continue
                entry["content"] = filtered
        # 仅在消息缺少时间戳时补充，保留已有来源时间。
        entry.setdefault("timestamp", datetime.now().isoformat())
        # 将清理后的消息追加到会话对象；外围流程再提交会话状态。
        session.messages.append(entry)
        if role == "user" and isinstance(followup_id, str) and followup_id:
            saved_followup_ids.add(followup_id)
        if role == "assistant":
            last_assistant_idx = len(session.messages) - 1
            declared_tool_call_ids.update(
                str(tc["id"])
                for tc_value in cast(
                    Iterable[object],
                    entry.get("tool_calls") or [],
                )
                if isinstance(tc_value, dict)
                for tc in (cast(dict[str, Any], tc_value),)
                if tc.get("id")
            )
    if summary_checkpoint is not None and checkpoint_boundary == len(messages):
        # 在原始 transcript 对应边界插入摘要检查点。
        session.commit_summary_checkpoint(summary_checkpoint.summary)
    if turn_latency_ms is not None and last_assistant_idx is not None:
        session.messages[last_assistant_idx]["latency_ms"] = int(turn_latency_ms)
    session.consumed_followup_ids |= saved_followup_ids
    session.updated_at = datetime.now()
```

</details>

**关键点解读：**

1. `skip` 是消息下标，不是字符串长度。例如已有 10 条历史，本轮新增 4 条，`messages[10:]` 取新增消息，不会把每条内容切成前 10 个字。
2. 保存的不只有最终答案，还包含后续重放所需的用户、assistant 和合法工具消息。工具结果必须对应已声明的调用 ID。
3. 此方法修改会话对象；轮次提交由外围持久化流程交给 `SessionState`，最终存储与 SQLite 配合。不能把“追加到列表”直接等同于“数据库事务已完成”。
4. 清理媒体块、工具输出预算、上下文压缩是不同阶段的策略。本方法没有旧示例的 `msg.text[:200]`、`response[:500]`，但也不能据此推断所有输入字节都原样保存。

### 面试要点

试着用自己的话解释：Loop 持续接收输入，按会话协调执行；Runner 对象复用，每次 run 维护执行上下文；保存阶段保留新增结构化消息并由会话层提交。能说清这些关系，比背方法长度更有用。

<a id="section-4-3"></a>

## 4.3 AgentRunner - runner.py

### 文件定位

当 Loop 已经准备好一轮输入后，Runner 决定“继续问模型、执行工具，还是结束”。本节沿用原教程的 ReAct 主题。

源码：[nanobot/agent/runner.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/runner.py#L284)，`AgentRunner.run`。

### 核心类：AgentRunner

`AgentRunSpec` 携带本轮执行条件，`AgentRunResult` 汇总本轮结果。`run()` 中创建本轮消息和 Hook 上下文，这是理解对象复用的关键：

<a id="method-agentrunner-run"></a>

源码：[nanobot/agent/runner.py:284](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/runner.py#L284)，`AgentRunner.run` 完整方法。

```python
# 管理一次 Agent 执行的生命周期，初始化本轮状态并在成功、失败或取消后收尾。
async def run(self, spec: AgentRunSpec) -> AgentRunResult:
    # 优先使用调用方传入的 Hook，否则使用基础生命周期实现。
    hook = spec.hook or AgentHook()
    has_run_hooks = hook.has_run_hooks()
    # 初始化本次运行的局部消息列表，避免把对象复用误解成共享消息历史。
    messages: list[dict[str, Any]] = []
    # 创建单次运行的 Hook 状态，记录输出、用量与停止原因。
    context = AgentRunHookContext(messages=[])
    llm_usage_source_token = bind_llm_usage_source(
        spec.llm_usage_source or source_from_session_key(spec.session_key)
    )

    # 包含 finally 清理路径，保证正常结束和异常退出都按约定收尾。
    try:
        # 包含 finally 清理路径，保证正常结束和异常退出都按约定收尾。
        try:
            try:
                # 准备初始消息与上下文压缩状态，再进入生命周期和核心迭代。
                messages, compaction = await self.prepare_prompt(self._initial_transcript_and_compaction, spec)
                context.messages = await self._run_context_snapshot(hook, messages) if has_run_hooks else []
                # 在核心执行前触发运行级扩展点。
                await hook.before_run(context)
                # 调用完整模型工具循环，结果包含消息、文本、用量和停止原因。
                result = await self._run_core(spec, hook, messages, compaction)
            # 处理 Exception，按本方法约定记录、转换或继续传播错误。
            except Exception as exc:
                context.messages = await self._run_context_snapshot(hook, messages) if has_run_hooks else []
                context.stop_reason = "error"
                context.error = f"Error: {type(exc).__name__}: {exc}"
                context.exception = exc
                # 通知扩展层本轮失败信息，实际异常是否继续抛出由后续代码决定。
                await hook.on_error(context)
                raise
            else:
                context.messages = await self._run_context_snapshot(hook, result.messages) if has_run_hooks else []
                context.final_content = result.final_content
                context.tools_used = list(result.tools_used)
                context.usage = result.usage
                context.stop_reason = result.stop_reason
                context.error = result.error
                context.tool_events = deepcopy(result.tool_events)
                context.had_injections = result.had_injections
                context.exception = None
                if context.error is not None:
                    # 通知扩展层本轮失败信息，实际异常是否继续抛出由后续代码决定。
                    await hook.on_error(context)
                # 把成功路径的运行结果交给运行级扩展点。
                await hook.after_run(context)
                return result
        # 单独处理取消信号，避免把真实取消误当成普通运行错误。
        except asyncio.CancelledError as exc:
            context.stop_reason = "cancelled"
            context.error = None
            context.exception = exc
            raise
        # 无论正常返回、异常或取消，都进入这里完成资源与状态收尾。
        finally:
            if has_run_hooks:
                try:
                    if isinstance(context.exception, asyncio.CancelledError):
                        context.messages = await self._run_context_snapshot(hook, messages)
                    else:
                        context.messages = await self._run_context_snapshot(hook, messages)
                # 单独处理取消信号，避免把真实取消误当成普通运行错误。
                except asyncio.CancelledError as exc:
                    context.stop_reason = "cancelled"
                    context.error = None
                    context.exception = exc
                    context.messages = await self._run_context_snapshot(hook, messages)
                    raise
    # 无论正常返回、异常或取消，都进入这里完成资源与状态收尾。
    finally:
        # 包含 finally 清理路径，保证正常结束和异常退出都按约定收尾。
        try:
            if context.exception is None:
                # 运行退出时收尾；与成功完成后的 after_run 职责不同。
                await hook.on_finally(context)
            else:
                try:
                    # 运行退出时收尾；与成功完成后的 after_run 职责不同。
                    await hook.on_finally(context)
                # 处理 Exception，按本方法约定记录、转换或继续传播错误。
                except Exception:
                    logger.opt(exception=tool_log_content_allowed()).error(
                        "AgentHook.on_finally error after {}",
                        context.stop_reason or "run exception",
                    )
        # 无论正常返回、异常或取消，都进入这里完成资源与状态收尾。
        finally:
            # 恢复当前异步执行上下文中的用量来源绑定。
            reset_llm_usage_source(llm_usage_source_token)
```

**关键点解读：**`messages` 和 `context` 是这次调用的局部变量；`spec.hook` 可以提供本轮扩展行为，缺省时使用基础 Hook。不要看到共享 Runner 就认定不同会话会共用同一个局部列表，也不要据此忽略工具实例和会话层自己的并发约束。

### run() 方法：ReAct 循环核心

新版把运行生命周期留在 `run()`，把迭代主体拆到 `_run_core()`。读原来的 run 主题时，需要顺着这个调用走一步。

完整的 `AgentRunner.run()` 已在[本章前文](#method-agentrunner-run)列出，可结合此处讲解对照阅读。

**关键点解读：**

1. ReAct 在这里体现为“模型选择行动 → 工具返回观察 → 模型继续判断”，不是固定交替输出某种思维文本。
2. 工具调用先以 assistant 消息记录，再追加带调用 ID 的结果，下一次模型请求才能知道工具查到了什么。
3. 没有工具调用时通常进入最终输出路径，但新版还会处理输入注入、继续和收尾策略，不能用一个无条件 `break` 概括所有分支。
4. 预算限制用于防止无休止循环；停止原因比“返回一段字符串”更丰富，调用方还需要区分正常完成、取消和错误。

**例子：**用户问“README 的安装步骤是什么”。模型先调用读取文件工具，工具返回内容，模型在下一次请求中依据内容组织答案。工具读到文件的那一步不等于最终回答已经完成。

### _execute_tool() 方法：工具执行（含 save_memory 拦截）

原教程把工具执行及 `save_memory` 特判写在此处。新版没有照搬这套私有方法，应读 `tools/execution.py` 与 `ToolRegistry.execute()`。原问题保留，答案改为实际分层：Runner 组织调用，执行层处理调度，Registry 校验并调用工具。

<a id="method-toolregistry-execute"></a>

源码：[nanobot/agent/tools/registry.py:197](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/registry.py#L197)，`ToolRegistry.execute` 完整方法。

```python
# 先准备和校验参数，再执行工具，把工具失败和异常转换成统一错误结果。
async def execute(self, name: str, params: Any) -> Any:
    """Execute a tool by name with given parameters."""
    # 执行前先完成工具查找、参数转换与校验。
    tool, params, error = self.prepare_call(name, params)
    if error:
        return ToolResult.error(with_tool_error_hint(str(error)))

    try:
        assert tool is not None  # guarded by prepare_call()
        # 只有通过准备阶段后才执行真实副作用。
        result = await tool.execute(**params)
        if is_tool_error_result(result):
            return ToolResult.error(
                with_tool_error_hint(str(result)),
                error_code=result.error_code if isinstance(result, ToolResult) else None,
            )
        return result
    # 处理 Exception，按本方法约定记录、转换或继续传播错误。
    except Exception as e:
        return ToolResult.error(with_tool_error_hint(f"Error executing {name}: {str(e)}"))
```

**关键点解读：**模型产生的是调用意图，Python 才执行实际副作用；参数 Schema 告诉模型怎么调用，运行时校验则检查真实参数。长期记忆读写及整理要沿当前 MemoryStore、工具和记忆流程追踪，不能继续解释成 Runner 必定拦截一个虚拟 `save_memory`。

### Lifecycle Hooks 详解

源码：[nanobot/agent/hook.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/hook.py#L67)，`AgentHook`。

| 时机 | 重点接口 | 阅读时要问的问题 |
| --- | --- | --- |
| 一次运行前后 | `before_run`、`after_run` | 整轮开始与结果完成时附加什么行为？ |
| 每次迭代 | `before_iteration`、`after_iteration` | 这一轮模型和工具执行处于什么状态？ |
| 错误和退出 | `on_error`、`on_finally` | 错误如何上报，退出如何收尾？ |

**关键点解读：**Hook 是执行到某个时机时调用的扩展入口。它把进度、用量和事件等附加行为从循环中抽出。`after_run` 与 `on_finally` 不等价：成功结果处理与退出收尾的职责不同，取消和异常也不能都当作正常回答。

### 关键设计决策

第一，Loop 与 Runner 分工，避免把平台收发写进模型循环。第二，执行输入和结果有明确类型，调用方可以判断停止原因。第三，工具通过统一注册与执行契约接入，Hook 通过生命周期接入。读完后，应能解释一次“调用工具后继续请求模型”的消息变化，而不只是记住类名。

<a id="section-4-4"></a>

## 4.4 ContextBuilder - context.py

### 文件定位

ContextBuilder 决定模型这一次看见什么。用户消息只是输入的一部分，身份、项目说明、历史、技能和记忆也会影响结果。

源码：[nanobot/agent/context.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/context.py#L75)，`ContextBuilder`。

### 核心类：ContextBuilder

先区分两个产物：系统提示词规定身份、规则和可用说明；消息序列表达历史及本次输入。把它们混成一个字符串，会看不清角色和工具调用关系。

### build_system_prompt() 方法

这个方法的主线是按条件收集 `parts`，最后连接。下面展示完整方法，并在各类上下文来源处加入注释：

<a id="method-contextbuilder-build-system-prompt"></a>

源码：[nanobot/agent/context.py:87](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/context.py#L87)，`ContextBuilder.build_system_prompt` 完整方法。

```python
# 按身份、项目说明、工具契约、记忆、技能和会话摘要组装系统提示词。
def build_system_prompt(
    self,
    *,
    channel: str | None = None,
    session_summary: SessionSummary | None = None,
    workspace: Path | None = None,
    include_memory: bool = True,
) -> str:
    """Build the system prompt from identity, bootstrap files, memory, and skills."""
    # 本轮指定的项目优先，否则使用默认工作区。
    root = workspace or self.workspace
    # 系统提示词从身份开始，再逐项加入资料。
    parts = [self._get_identity(channel=channel, workspace=root)]

    # 加载启动说明，区别于聊天历史。
    bootstrap = self._load_bootstrap_files(root)
    if bootstrap:
        parts.append(bootstrap)

    parts.append(render_template("agent/tool_contract.md", track_source=True))

    project_path = root.expanduser().resolve()
    if project_path != self.workspace.expanduser().resolve():
        parts.append(
            "# Current Project\n\n"
            f"Working directory: {project_path}\n"
            "Use it as the default root for project files and relative tool paths."
        )

    # 本轮允许加载长期记忆时才读取。
    if include_memory:
        memory = self.memory.read_memory()
        if memory and not self._is_template_content(memory, "memory/MEMORY.md"):
            parts.append(memory)
            record_source(self.memory.memory_file, memory)
        elif self.memory.memory_file.is_file():
            record_source(self.memory.memory_file, "", loaded=False)

    # 本次组装使用一致的技能发现与元数据快照。
    with self.skills.snapshot():
        # always 技能加载正文，其余技能随后只先给摘要。
        active_skills = self.skills.get_always_skills()
        if active_skills:
            active_content = self.skills.load_skills_for_context(active_skills)
            if active_content:
                parts.append(f"# Active Skills\n\n{active_content}")

        skills_summary = self.skills.build_skills_summary(
            exclude=set(active_skills),
            workspace=root,
        )
    if skills_summary:
        parts.append(render_template("agent/skills_section.md", skills_summary=skills_summary, track_source=True))

    if session_summary and session_summary["text"] != "(nothing)":
        parts.append(
            "[Archived Context Summary]\n\n"
            f"Previous conversation summary (last active {session_summary['last_active']}):\n"
            f"{session_summary['text']}"
        )

    return "\n\n---\n\n".join(parts)
```

长期记忆加载分支也已包含在上述完整方法中：

完整的 `ContextBuilder.build_system_prompt()` 已在[本章前文](#method-contextbuilder-build-system-prompt)列出，可结合此处讲解对照阅读。

**关键点解读：**`workspace or self.workspace` 允许本轮指定项目；bootstrap 是启动说明来源，不能与消息历史混同。记忆和技能有加载条件，因此“磁盘上有文件”不等于“每次都全文注入”。always 技能加载正文，其余技能先提供摘要，体现按需获取知识。

### build_messages() 方法

新版中它是兼容包装器，先构造 `TranscriptInput` 并调用 `build_transcript()`，然后按条件合并相邻同角色消息。核心运行路径阅读时还要进入 transcript 构建，而不是只盯这个旧入口名。

<a id="method-contextbuilder-build-messages"></a>

源码：[nanobot/agent/context.py:223](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/context.py#L223)，`ContextBuilder.build_messages` 完整方法。

```python
# 把输入交给 transcript 构建器，再处理相邻同角色消息的兼容合并。
def build_messages(
    self,
    history: list[dict[str, Any]],
    current_message: str | None,
    *,
    media: list[str] | None = None,
    channel: str | None = None,
    current_role: str = "user",
    session_summary: SessionSummary | None = None,
    runtime_context_blocks: Sequence[RuntimeContextBlock] | None = None,
    workspace: Path | None = None,
    include_memory: bool = True,
) -> list[dict[str, Any]]:
    """Compatibility wrapper for callers that need merged adjacent roles."""
    # 兼容入口委托给真正的 transcript 构建器。
    messages = self.build_transcript(
        TranscriptInput(
            history=history,
            current_message=current_message,
            media=media,
            current_role=current_role,
            session_summary=session_summary,
            runtime_context_blocks=runtime_context_blocks,
        ),
        channel=channel,
        workspace=workspace,
        include_memory=include_memory,
    )
    if current_message is None:
        return messages
    current = messages[-1]
    if len(messages) < 2 or messages[-2].get("role") != current.get("role"):
        return messages

    # 复制前一条消息后合并，避免直接就地改写原条目。
    merged = dict(messages[-2])
    merged["content"] = self._merge_message_content(
        merged.get("content"),
        current.get("content"),
    )
    current_meta = current.get("_meta")
    if current.get("role") == "user" and isinstance(current_meta, dict):
        internal_meta = dict(merged.get("_meta") or {})
        internal_meta.update(cast(dict[str, Any], current_meta))
        merged["_meta"] = internal_meta
    return [*messages[:-2], merged]
```

**关键点解读：**历史消息保留角色与工具调用结构；当前用户消息追加在本轮输入位置；运行时上下文是另一类输入。调试回答遗漏信息时，先检查最终 transcript 里是否真的包含它，再判断是模型问题还是上下文装配问题。

### Prompt Cache 优化策略解读

固定身份与说明通常比每轮用户输入稳定，把稳定内容放在前面有利于兼容服务复用共同前缀。实际缓存命中仍取决于 Provider、模型和请求格式；这里的组装顺序本身不能保证一定命中缓存。学习时比较连续两轮请求：哪些前缀保持一致，哪些部分必然变化？

### 面试要点

能解释 system prompt 与 transcript 的区别；能说明技能摘要和正文为何分开加载；能沿 `build_messages → build_transcript` 找到真正的消息装配位置。

<a id="section-4-5"></a>

## 4.5 MemoryStore - memory.py

### 文件定位

本节仍然回答原教程的三个问题：长期记忆怎么读、怎么写，历史增长后怎么处理。需要先把长期知识、结构化会话、可检索历史分开。

源码：[nanobot/agent/memory.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L52)，`MemoryStore`。

### 核心类：MemoryStore

| 内容 | 当前主要位置 | 用途 |
| --- | --- | --- |
| 长期记忆 | `memory/MEMORY.md` | 跨轮复用的整理知识 |
| 历史条目 | `memory/history.jsonl` | 逐条追加的可检索记录，旧 `HISTORY.md` 有迁移路径 |
| 会话消息 | Session / SessionState / SQLite | 恢复对话和工具调用关系 |

**关键点解读：**它们服务不同的读取方式。不要把 `_save_turn()` 保存会话、`append_history()` 追加条目、长期记忆整理说成同一个操作。

### read_memory() 方法

<a id="method-memorystore-read-memory"></a>

源码：[nanobot/agent/memory.py:221](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L221)，`MemoryStore.read_memory` 完整方法。

```python
# 读取长期记忆文本；是否注入模型由上下文构建器决定。
def read_memory(self) -> str:
    return self.read_file(self.memory_file)
```

**关键点解读：**读取目标由 `self.memory_file` 决定，返回文本后是否放进系统提示词由 ContextBuilder 的加载条件决定。文件读取与提示词注入是两个步骤。

### save_memory() 方法

原教程的“保存长期记忆”主题保留；新版实际方法名是 `write_memory()`：

<a id="method-memorystore-write-memory"></a>

源码：[nanobot/agent/memory.py:224](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L224)，`MemoryStore.write_memory` 完整方法。

```python
# 将给定文本写入长期记忆文件，不负责自动挑选值得保存的知识。
def write_memory(self, content: str) -> None:
    self.memory_file.write_text(content, encoding="utf-8")
```

**关键点解读：**这是把给定文本写到记忆文件的存储动作，不负责自己判断什么值得记住，也不等于模型天然拥有可靠长期记忆。内容选择、整理和后续注入仍由外围流程承担。

### append_history() 方法

<a id="method-memorystore-append-history"></a>

源码：[nanobot/agent/memory.py:269](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L269)，`MemoryStore.append_history` 完整方法。

```python
# 清理历史条目并应用长度预算，再交给受锁保护的历史写入方法。
def append_history(
    self,
    entry: str,
    *,
    max_chars: int | None = None,
    session_key: str | None = None,
) -> int:
    """Append *entry* to history.jsonl and return its auto-incrementing cursor.

    Entries are passed through `strip_think` to drop template-level leaks
    (e.g. unclosed `<think` prefixes, `<channel|>` markers) before being
    persisted. If the cleaned content is empty but the raw entry wasn't,
    the record is persisted with an empty string rather than falling back
    to the raw leak — otherwise `strip_think`'s guarantees would be
    undone when Dream consumes the journal entry.

    A defensive cap (*max_chars*, default ``_HISTORY_ENTRY_HARD_CAP``) is
    applied as a final safety net: individual callers should cap their own
    content more tightly; this default only exists to catch unintentional
    large writes (e.g. an LLM echoing its input back as a "summary").
    """
    # 先清理可能泄漏的模板内容并应用条目长度预算。
    content = self._normalize_history_entry(entry, max_chars=max_chars)
    if entry.rstrip() and not content:
        logger.debug(
            "history entry stripped to empty (likely template leak); "
            "persisting empty content to avoid re-polluting Dream input",
        )
    # 把已清理内容交给原子追加入口，并返回历史游标。
    return self._append_history_record(content, session_key=session_key)
```

`append_history()` 委托的历史追加方法如下：

<a id="method-memorystore-append-history-record"></a>

源码：[nanobot/agent/memory.py:298](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L298)，`MemoryStore._append_history_record` 完整方法。

```python
# 原子分配历史游标并追加一行 JSON，防止并发写入重复游标。
def _append_history_record(self, content: str, *, session_key: str | None = None) -> int:
    """Persist already sanitized, bounded content without rewriting chunk edges."""
    ts = datetime.now().strftime("%Y-%m-%d %H:%M")
    # Cursor allocation and the append must be atomic: concurrent writers
    # could otherwise read the same current cursor and emit duplicates.
    # 游标分配与追加写入在同一把锁内完成。
    with self._append_lock:
        # 分配新的递增位置，供后续历史整理跟踪进度。
        cursor = self._next_cursor()
        record = {"cursor": cursor, "timestamp": ts, "content": content}
        if session_key:
            record["session_key"] = session_key
        # 追加写入 JSONL，每条历史占一行。
        with open(self.history_file, "a", encoding="utf-8") as f:
            f.write(json.dumps(record, ensure_ascii=False) + "\n")
        # 保存当前游标，防止下次分配重复位置。
        self._cursor_file.write_text(str(cursor), encoding="utf-8")
    return cursor
```

**关键点解读：**当前入口接收字符串条目并返回游标，且是同步方法，不是旧例子中 `await append_history(turn_summary)`。JSONL 便于逐条读取；游标用于定位整理进度，不能与截取用户前多少字符混为一谈。具体长度预算应查看该方法参数和调用方。

### MemoryConsolidator 压缩机制

源码：[nanobot/agent/memory.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L1034)，`Consolidator`。

原小节标题中的 MemoryConsolidator 在新版应对应到 `Consolidator`。要联合理解它与 `MemoryArchiver` 和上下文治理：归档负责历史整理的一部分，Consolidator 提供摘要能力，ContextGovernor 关注请求上下文预算。它们协作，不能再用旧的“消息达到固定 N 条统一保存”代替。

**关键点解读：**压缩用较短文本保留后续需要的信息，必然涉及取舍；原始会话存储与当前模型能看到的上下文不同。排查“忘记某件事”时，依次检查原始会话是否保存、摘要是否保留、当前请求是否加载，才能定位信息在哪一层丢失。

<a id="section-4-6"></a>

## 4.6 SubagentManager - subagent.py

### 文件定位

子 Agent 用于把一项工作交给独立执行路径，管理器负责创建、排队、状态与回传。

源码：[nanobot/agent/subagent.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/subagent.py#L206)，`SubagentManager`。

### 核心类：SubagentManager

子任务管理器也有自己的 Runner 与任务调度状态。因此“主 Loop 复用一个 Runner”不能扩展成“整个进程绝对只有一个 Runner”。主对话和子任务有各自的生命周期。

### spawn() 方法

<a id="method-subagentmanager-spawn"></a>

源码：[nanobot/agent/subagent.py:609](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/subagent.py#L609)，`SubagentManager.spawn` 完整方法。

```python
# 创建有来源信息的子任务，返回任务状态和标识；创建回执不等于执行完成。
async def spawn(
    self,
    task: str,
    label: str | None = None,
    origin_channel: str = "cli",
    origin_chat_id: str = "direct",
    session_key: str | None = None,
    origin_message_id: str | None = None,
    temperature: float | None = None,
    workspace_scope: WorkspaceScope | None = None,
    *,
    runtime: LLMRuntime | None = None,
    origin_turn_id: str | None = None,
    session_policy: SessionPolicy | None = None,
) -> str:
    """Start background work and route its terminal result to the parent."""
    try:
        # 创建子任务记录并安排执行，保留来源与运行条件。
        record = await self._create_task(
            task, label, origin_channel, origin_chat_id, session_key,
            origin_message_id, temperature, workspace_scope, runtime, announce=True, origin_turn_id=origin_turn_id,
            session_policy=session_policy,
        )
    # 处理 SubagentControlError，按本方法约定记录、转换或继续传播错误。
    except SubagentControlError as exc:
        return ToolResult.error(f"Error: {exc}")
    status = record.status
    return f"Subagent [{status.label}] started (id: {status.task_id}). I'll notify you when it completes."
```

**关键点解读：**创建任务不代表立即完成；达到并发上限后需要排队；取消、查询与结果回传都需要明确的任务标识。当前配置默认并发子任务数为 4，迭代预算来自配置，不要沿用旧示例固定 15 轮的说法。

**例子：**主任务让子任务分别阅读两个文件，创建后可以继续处理协调工作，最终根据任务状态取得结果。不能把一次创建返回值当作两个文件都已分析完毕。

<a id="section-4-7"></a>

## 4.7 工具系统 - tools/

### 4.7.1 ToolRegistry - registry.py

Registry 是“名称到工具实例”的索引，同时承担调用准备与执行入口。注册不是执行；模型看到的定义也不是 Python 函数本身。

完整的 `ToolRegistry.execute()` 已在[本章前文](#method-toolregistry-execute)列出，可结合此处讲解对照阅读。

**关键点解读：**`prepare_call` 查找工具并规范、校验参数；错误先返回 `ToolResult.error`；只有通过后才进入 `execute`。源码后半段还处理工具返回的错误及异常。学习重点是“校验在副作用前”，不是背错误提示文字。

### 4.7.2 MCPToolWrapper - mcp.py

<a id="method-mcptoolwrapper-execute"></a>

源码：[nanobot/agent/tools/mcp.py:743](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/mcp.py#L743)，`MCPToolWrapper.execute` 完整方法。

```python
# 通过已管理的 MCP 会话调用远端工具，渲染结果并保留错误语义。
async def execute(self, **kwargs: Any) -> str:
    # 使用 MCP 管理的会话、超时与恢复路径调用远端工具。
    result = await self._request_with_recovery(
        lambda session: session.call_tool(self._original_name, arguments=kwargs),
        self._tool_timeout,
        "tool",
    )
    if isinstance(result, ToolResult):
        return result
    try:
        rendered = self._render_call_result(result.content, kwargs)
        # 保持远端错误语义，不把错误响应误当成正常文本。
        if getattr(result, "isError", False):
            return ToolResult.error(rendered)
        return rendered
    # 处理 Exception，按本方法约定记录、转换或继续传播错误。
    except Exception as exc:
        logger.opt(exception=tool_log_content_allowed()).error(
            "MCP tool '{}' failed while rendering result: {}: {}",
            self._name,
            type(exc).__name__,
            exc if tool_log_content_allowed() else "[content hidden]",
        )
        return ToolResult.error(
            f"(MCP tool returned malformed content: {type(exc).__name__})"
        )
```

**关键点解读：**包装器让 Runner 不必分别理解每个远端服务；连接生命周期由 Provider 管理，不能每次工具调用临时拼一个裸 HTTP 请求。远端执行返回错误也需要保持错误语义，不能把它当成功文本。

### 4.7.3 MessageTool - message.py

源码：[nanobot/agent/tools/message.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/message.py#L141)，`MessageTool.execute`。

MessageTool 允许模型主动发送消息。读源码时跟踪三件事：目标来自参数还是本轮上下文，内容怎样变成出站消息，交付结果怎样反馈给工具调用方。

**关键点解读：**工具发送消息属于一次副作用，Runner 的最终回复属于轮次输出，两者不是同一个动作。实现还需记录已发送内容等状态，避免把主动发送与最终交付混为一谈。不要在客户端重写这套后端交付规则。


<details>
<summary>展开 MessageTool.execute() 完整方法（120 行源码）</summary>

<a id="method-messagetool-execute"></a>

源码：[nanobot/agent/tools/message.py:141](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/message.py#L141)，`MessageTool.execute` 完整方法。

```python
# 根据本轮上下文和参数发送主动消息，并记录发送结果。
async def execute(
    self,
    content: str,
    channel: str | None = None,
    chat_id: str | None = None,
    message_id: str | None = None,
    media: list[str] | None = None,
    buttons: Any = None,
    **kwargs: Any,
) -> str:
    from nanobot.utils.helpers import strip_think

    content = strip_think(content)

    button_rows: list[list[str]] | None = None
    if buttons is not None:
        raw_buttons = cast(list[Any], buttons) if isinstance(buttons, list) else None
        if raw_buttons is None or any(
            not isinstance(row, list)
            or any(not isinstance(label, str) for label in cast(list[Any], row))
            for row in raw_buttons
        ):
            return ToolResult.error("Error: buttons must be a list of list of strings")
        button_rows = cast(list[list[str]], raw_buttons)
    request_ctx = current_request_context()
    default_channel = (
        request_ctx.channel if request_ctx is not None else self._fallback_channel
    )
    default_chat_id = (
        request_ctx.chat_id if request_ctx is not None else self._fallback_chat_id
    )
    default_message_id = (
        request_ctx.message_id
        if request_ctx is not None
        else self._fallback_message_id
    )
    default_metadata = (
        request_ctx.metadata
        if request_ctx is not None
        else self._fallback_metadata
    )
    channel = channel or default_channel
    explicit_chat_id = chat_id
    if (
        default_channel == "websocket"
        and channel == "websocket"
        and explicit_chat_id is not None
        and str(explicit_chat_id).strip() != ""
        and str(explicit_chat_id).strip() != str(default_chat_id).strip()
    ):
        return ToolResult.error(
            "Error: chat_id does not match the active WebSocket conversation. "
            "Omit chat_id (and usually channel) so delivery uses the current "
            "conversation id from context — WebSocket client_id strings "
            "(e.g. anon-…) are not chat ids."
        )
    chat_id = chat_id or default_chat_id
    # Only inherit default message_id when targeting the same channel+chat.
    # Cross-chat sends must not carry the original message_id, because
    # some channels (e.g. Feishu) use it to determine the target
    # conversation via their Reply API, which would route the message
    # to the wrong chat entirely.
    same_target = channel == default_channel and chat_id == default_chat_id
    if same_target:
        message_id = message_id or default_message_id
    else:
        message_id = None

    if not channel or not chat_id:
        return ToolResult.error("Error: No target channel/chat specified")

    if not self._send_callback:
        return ToolResult.error("Error: Message sending not configured")

    if media:
        try:
            media = self._resolve_media(media)
        # 处理 (OSError, PermissionError, ValueError)，按本方法约定记录、转换或继续传播错误。
        except (OSError, PermissionError, ValueError) as e:
            return ToolResult.error(f"Error: media path is not allowed: {str(e)}")

    metadata = dict(default_metadata) if same_target else {}
    if message_id:
        metadata["message_id"] = message_id
    if media:
        metadata["_record_channel_delivery"] = True
    if same_target and request_ctx is not None and request_ctx.session_generation is not None:
        metadata["_session_key"] = request_ctx.session_key
        metadata["_session_generation"] = request_ctx.session_generation

    msg = OutboundMessage(
        channel=channel,
        chat_id=chat_id,
        content=content,
        media=media or [],
        buttons=button_rows or [],
        metadata=metadata,
        delivery=(DeliveryRequest.from_context(
            default_metadata, request_ctx.session_key, request_ctx.session_generation,
        ) if request_ctx is not None else None),
    )

    if self._suppress_delivery_var.get():
        logger.debug("MessageTool: delivery suppressed during internal check")
        return f"Message acknowledged for {channel}:{chat_id} (not delivered)"

    try:
        await self._send_callback(msg)
        sends = _CURRENT_MESSAGE_SENDS.get()
        if sends is not None:
            sends.add((channel, chat_id))
        media_info = f" with {len(media)} attachments" if media else ""
        button_info = (
            f" with {sum(len(row) for row in button_rows)} button(s)"
            if button_rows
            else ""
        )
        action = "queued" if msg.delivery is not None else "sent"
        return f"Message {action} to {channel}:{chat_id}{media_info}{button_info}"
    # 处理 Exception，按本方法约定记录、转换或继续传播错误。
    except Exception as e:
        return ToolResult.error(f"Error sending message: {str(e)}")
```

</details>

### 4.7.4 SpawnTool - spawn.py

这里保留原教程“用工具创建子任务”的位置。新版文件为 `tools/subagent.py`，类为 `SubagentTool`，工具名为 `subagent`。

<a id="method-subagenttool-execute"></a>

源码：[nanobot/agent/tools/subagent.py:86](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/subagent.py#L86)，`SubagentTool.execute` 完整方法。

```python
# 按动作创建、查询、发送输入或取消子任务，并核对任务归属。
async def execute(
    self, action: str, task_id: str | None = None, message: str | None = None,
    task: str | None = None, label: str | None = None,
    temperature: float | None = None, wait: bool = False, **kwargs: Any,
) -> str:
    # 创建动作需要非空任务说明；其他动作管理已有任务。
    if action == "create":
        if not task or not task.strip():
            return ToolResult.error("Error: create requires a non-empty task")
        return await self._create_task(task, label, temperature, wait)
    # 取得调用会话身份，用于查询、发送和取消时核对任务归属。
    owner = current_request_session_key()
    try:
        if action == "check":
            if not owner:
                raise SubagentControlError("task unavailable")
            if task_id is not None:
                return json.dumps(self._manager.check(task_id, owner).as_dict(), ensure_ascii=False)
            return json.dumps({
                "tasks": [status.as_dict()
                          for status in self._manager.statuses_for_session(owner, include_history=False).values()],
            }, ensure_ascii=False)
        if action in {"send", "cancel"}:
            if not task_id:
                return ToolResult.error(f"Error: {action} requires task_id")
            if action == "send":
                return json.dumps(await self._manager.send(task_id, owner, message))
            status = await self._manager.cancel(task_id, owner)
            return json.dumps(status.as_dict(), ensure_ascii=False)
        return ToolResult.error("Error: unknown subagent action")
    # 处理 SubagentControlError，按本方法约定记录、转换或继续传播错误。
    except SubagentControlError as exc:
        return ToolResult.error(f"Error: {exc}")
```

**关键点解读：**模型通过统一工具接口提出管理任务的请求，SubagentManager 才负责真实调度。新版已不只是一个 `spawn(task)` 动作，阅读时要分清工具层的动作分派与管理器层的任务执行。

<a id="section-4-8"></a>

## 4.8 MessageBus - bus/

### 核心类：MessageBus

消息总线把平台输入与 Agent 执行解耦。新版源码位于 `bus/queue.py`，输入消息结构在 `bus/events.py`。

<a id="method-messagebus-consume-inbound"></a>

源码：[nanobot/bus/queue.py:44](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/bus/queue.py#L44)，`MessageBus.consume_inbound` 完整方法。

```python
# 从统一输入队列异步取出一条消息。
async def consume_inbound(self) -> InboundMessage:
    """Consume the next inbound message (blocks until available)."""
    # 等待并取走一条输入，不在总线中执行模型循环。
    return await self.inbound.get()
```

**关键点解读：**没有消息时 `get()` 异步等待；有消息时消费方取出继续处理。渠道只需按契约发布输入，不必知道 Runner 如何迭代。

新版事件还要区分：`publish` 等待本地订阅者处理状态事件，`publish_event` 把需要渠道路由的事件放入交付队列。它们不能都解释成“立即把回复发送到网络”。队列入队、事件处理、实际平台发送是不同完成时刻。

<a id="section-4-9"></a>

## 4.9 Channel 适配层 - channels/

### 基类：BaseChannel

源码：[nanobot/channels/base.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/channels/base.py#L26)，`BaseChannel`。

BaseChannel 定义启动、停止和发送等共享契约。平台 SDK、消息字段与地址格式由具体渠道适配；执行、会话和安全策略仍属于 Python 后端。

### Telegram 适配器示例

<a id="method-telegramchannel-on-message"></a>

源码：[nanobot/channels/telegram/runtime.py:1943](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/channels/telegram/runtime.py#L1943)，`TelegramChannel._on_message` 完整方法。

```python
# 校验平台更新并进入有序处理路径，避免消息处理乱序。
async def _on_message(self, update: Update, context: ContextTypes.DEFAULT_TYPE) -> None:
    if not update.message or not update.effective_user:
        return
    if not self._running:
        await self._process_message_update(update)
        return
    self._enqueue_ordered_update(kind="message", update=update)
```

**关键点解读：**一个平台群聊可能有话题，不能仅凭用户 ID 认定会话；媒体下载与模型执行也不是同一职责。阅读这个例子时，重点找“平台消息变成统一消息”的边界。

### 飞书适配器示例

源码：[nanobot/channels/feishu/runtime.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/channels/feishu/runtime.py#L2547)，`FeishuChannel._on_message`。

飞书同样把平台事件转换为统一输入，但需要处理自己的事件结构、提及、群聊与回复地址。与 Telegram 对照阅读：平台特有解析不同，进入消息总线之后共享的执行链路相同。新增平台的主要工作应落在适配层，而不是复制整个 ReAct 循环。

<details>
<summary>展开 FeishuChannel._on_message() 完整方法（206 行源码）</summary>

<a id="method-feishuchannel-on-message"></a>

源码：[nanobot/channels/feishu/runtime.py:2547](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/channels/feishu/runtime.py#L2547)，`FeishuChannel._on_message` 完整方法。

```python
# 解析飞书事件、会话地址与媒体，将平台输入交给统一渠道处理流程。
async def _on_message(self, data: P2ImMessageReceiveV1) -> None:
    if not self._running:
        return
    try:
        event = data.event
        if event is None or event.message is None or event.sender is None:
            self.logger.warning("Ignoring incomplete Feishu message event")
            return
        message = event.message
        sender = event.sender

        self.logger.debug("raw message: {}", message.content)
        self.logger.debug("mentions: {}", getattr(message, "mentions", None))

        message_id = message.message_id

        if sender.sender_type == "bot":
            return

        sender_id = sender.sender_id.open_id if sender.sender_id else "unknown"
        chat_id = message.chat_id
        chat_type = message.chat_type
        msg_type = message.message_type
        if not all(isinstance(value, str) and value for value in (
            message_id,
            sender_id,
            chat_id,
            chat_type,
            msg_type,
        )):
            self.logger.warning("Ignoring Feishu message event with missing routing fields")
            return
        message_id = cast(str, message_id)
        sender_id = cast(str, sender_id)
        chat_id = cast(str, chat_id)
        chat_type = cast(str, chat_type)
        msg_type = cast(str, msg_type)

        if chat_type == "group" and not self._is_group_message_for_bot(message):
            self.logger.debug("skipping group message (not mentioned)")
            return

        if message_id in self._processed_message_ids:
            return
        self._processed_message_ids[message_id] = None

        while len(self._processed_message_ids) > 1000:
            self._processed_message_ids.popitem(last=False)

        # Early permission check — avoid side effects for unauthorized users.
        # Group chats are silently ignored; DMs get a pairing code.
        if not self.is_allowed(sender_id):
            if chat_type == "p2p":
                # content="" because the pairing reply is generated by
                # BaseChannel._handle_message, not from the original message.
                await self._handle_message(
                    sender_id=sender_id,
                    chat_id=sender_id,
                    content="",
                    is_dm=True,
                )
            return

        task = asyncio.create_task(
            self._add_reaction(message_id, self.config.react_emoji)
        )
        self._background_tasks.add(task)
        task.add_done_callback(self._on_background_task_done)
        task.add_done_callback(lambda t: self._on_reaction_added(message_id, t))

        content_parts: list[str] = []
        media_paths: list[str] = []

        try:
            raw_content = message.content if isinstance(message.content, str) else ""
            content_json = _as_json_object(json.loads(raw_content)) if raw_content else {}
        # 处理 json.JSONDecodeError，按本方法约定记录、转换或继续传播错误。
        except json.JSONDecodeError:
            content_json = {}
        content_json = content_json or {}

        if msg_type == "text":
            text = content_json.get("text", "")
            if isinstance(text, str) and text:
                mentions = getattr(message, "mentions", None)
                text = self._strip_leading_bot_mention(text, mentions)
                text = self._resolve_mentions(text, mentions)
                content_parts.append(text)

        elif msg_type == "post":
            text, image_keys = _extract_post_content(content_json)
            if text:
                content_parts.append(text)
            for img_key in image_keys:
                file_path, content_text = await self._download_and_save_media(
                    "image", {"image_key": img_key}, message_id
                )
                if file_path:
                    media_paths.append(file_path)
                content_parts.append(content_text)
            # Some posts include files alongside the text and images.
            post_files, post_folders = _extract_post_attachments(content_json)
            for folder_name in post_folders:
                content_parts.append(
                    f"[folder: {folder_name} — folder attachments cannot be downloaded; "
                    "ask the user to send a zip archive]"
                )
            for entry in post_files:
                file_path, content_text = await self._download_and_save_media(
                    "file", entry, message_id
                )
                if file_path:
                    media_paths.append(file_path)
                content_parts.append(content_text)

        elif msg_type in ("image", "audio", "file", "media"):
            file_path, content_text = await self._download_and_save_media(
                msg_type, content_json, message_id
            )
            if file_path:
                media_paths.append(file_path)

            if msg_type == "audio" and file_path:
                transcription = await self.transcribe_audio(file_path)
                if transcription:
                    content_text = f"[transcription: {transcription}]"

            content_parts.append(content_text)

        elif msg_type in (
            "share_chat",
            "share_user",
            "interactive",
            "share_calendar_event",
            "system",
            "merge_forward",
        ):
            text = _extract_share_card_content(content_json, msg_type)
            if text:
                content_parts.append(text)

        else:
            content_parts.append(MSG_TYPE_MAP.get(msg_type, f"[{msg_type}]"))

        parent_id = getattr(message, "parent_id", None)
        root_id = getattr(message, "root_id", None)
        thread_id = getattr(message, "thread_id", None)
        parent_id = parent_id if isinstance(parent_id, str) else None
        root_id = root_id if isinstance(root_id, str) else None
        thread_id = thread_id if isinstance(thread_id, str) else None

        if parent_id and self._client:
            loop = asyncio.get_running_loop()
            reply_ctx = await loop.run_in_executor(
                None, self._get_message_content_sync, parent_id
            )
            if reply_ctx:
                content_parts.insert(0, reply_ctx)

        content = "\n".join(content_parts) if content_parts else ""

        if not content and not media_paths:
            return

        if chat_type == "p2p" and normalize_command_text(content).lower() == "/new":
            loop = asyncio.get_running_loop()
            await loop.run_in_executor(
                None,
                self._send_message_sync,
                "open_id",
                sender_id,
                "system",
                _NEW_SESSION_DIVIDER_CONTENT,
            )

        # Build session key for conversation isolation.
        # If topic_isolation is True: each topic gets its own session via root_id/message_id.
        # If topic_isolation is False: all messages in group share the same session.
        # Private chat: no override — same behavior as Telegram/Slack.
        if chat_type == "group":
            if self.config.topic_isolation:
                session_key = f"{self.name}:{chat_id}:{root_id or message_id}"
            else:
                session_key = f"{self.name}:{chat_id}"
        else:
            session_key = None

        reply_to = chat_id if chat_type == "group" else sender_id
        await self._handle_message(
            sender_id=sender_id,
            chat_id=reply_to,
            content=content,
            media=media_paths,
            metadata={
                "message_id": message_id,
                "chat_type": chat_type,
                "msg_type": msg_type,
                "parent_id": parent_id,
                "root_id": root_id,
                "thread_id": thread_id,
            },
            session_key=session_key,
            is_dm=chat_type == "p2p",
        )

    # 处理 Exception，按本方法约定记录、转换或继续传播错误。
    except Exception:
        self.logger.exception("Error processing message")
```

</details>

<a id="section-4-10"></a>

## 4.10 配置系统 - config/schema.py




### 配置数据类

源码：[nanobot/config/schema.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/config/schema.py#L479)，`Config`。

配置使用 Pydantic 模型表达类型、默认值和嵌套结构。原教程的 YAML 讲解改为当前 `config.json`，但“配置声明与配置加载分开”的学习重点不变。

**关键点解读：**先在 Schema 查字段是否合法，再到调用方查它何时生效。界面显示的模型预设、运行时选择的 Provider、工具自己的配置有各自的装配位置，不能只读一个扁平字段表。

### 配置加载

<a id="method-load-config"></a>

源码：[nanobot/config/loader.py:36](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/config/loader.py#L36)，`load_config` 完整方法。

```python
# 读取、迁移并校验配置，区分缺失、解析错误和结构错误。
def load_config(config_path: Path | None = None) -> Config:
    """
    Load configuration from file or create default.

    Args:
        config_path: Optional path to config file. Uses default if not provided.

    Returns:
        Loaded configuration object.
    """
    path = config_path or get_config_path()

    if not path.exists():
        try:
            config = Config()
        # 处理 SettingsError，按本方法约定记录、转换或继续传播错误。
        except SettingsError as exc:
            raise ConfigLoadError(
                path,
                kind="invalid_schema",
                summary=(
                    "Environment-based configuration could not be parsed. "
                    "Check that complex NANOBOT_* values use valid JSON."
                ),
            ) from exc
        # 处理 ValidationError，按本方法约定记录、转换或继续传播错误。
        except ValidationError as exc:
            raise ConfigLoadError(
                path,
                kind="invalid_schema",
                summary="Environment-based configuration is invalid.",
                issues=validation_issues(exc),
            ) from exc
        config.bind_source_path(path)
        _apply_ssrf_whitelist(config)
        return config

    try:
        with path.open(encoding="utf-8") as handle:
            data = json.load(handle)
    # 处理 json.JSONDecodeError，按本方法约定记录、转换或继续传播错误。
    except json.JSONDecodeError as exc:
        raise ConfigLoadError(
            path,
            kind="invalid_json",
            summary=(
                f"JSON syntax error at line {exc.lineno}, column {exc.colno}: "
                f"{_sentence(exc.msg)}"
            ),
        ) from exc
    # 处理 UnicodeDecodeError，按本方法约定记录、转换或继续传播错误。
    except UnicodeDecodeError as exc:
        raise ConfigLoadError(
            path,
            kind="io_error",
            summary="The file is not valid UTF-8.",
        ) from exc
    # 处理 OSError，按本方法约定记录、转换或继续传播错误。
    except OSError as exc:
        detail = exc.strerror or type(exc).__name__
        raise ConfigLoadError(
            path,
            kind="io_error",
            summary=f"Unable to read the file: {_sentence(detail)}",
        ) from exc

    if not isinstance(data, dict):
        root_type = type(data).__name__
        raise ConfigLoadError(
            path,
            kind="invalid_root",
            summary="The top level of config.json must be a JSON object.",
            issues=(
                ConfigIssue(
                    path=(),
                    message=f"Expected an object, but found {root_type}.",
                ),
            ),
        )

    # 迁移旧字段后再用当前 Schema 校验。
    data = _migrate_config(cast(dict[str, Any], data))
    try:
        # JSON 可解析不代表配置有效；这里校验类型和字段约束。
        config = Config.model_validate(data)
    # 处理 ValidationError，按本方法约定记录、转换或继续传播错误。
    except ValidationError as exc:
        issues = validation_issues(exc)
        raise ConfigLoadError(
            path,
            kind="invalid_schema",
            summary=f"Found {len(issues)} invalid setting(s).",
            issues=issues,
        ) from exc

    config.bind_source_path(path)
    _apply_ssrf_whitelist(config)
    return config
```

**关键点解读：**JSON 能解析只证明语法有效，Pydantic 校验通过才说明满足配置结构；API Key 可用、模型服务可达仍需实际请求验证。`${VAR}` 是配置环境变量引用语法，不是把 PowerShell 的 `$env:VAR` 原样填进 JSON。

<a id="section-4-11"></a>

## 4.11 关键代码片段解读

### 片段 1：完整的消息处理链路

<details>
<summary>展开 AgentLoop._process_message() 完整方法（108 行源码）</summary>

<a id="method-agentloop-process-message"></a>

源码：[nanobot/agent/loop.py:1844](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/loop.py#L1844)，`AgentLoop._process_message` 完整方法。

```python
# 为一条输入创建轮次上下文，顺序执行恢复、压缩、命令、构建、运行、保存和回复。
async def _process_message(
    self,
    msg: InboundMessage,
    session_key: str | None = None,
    on_progress: Callable[..., Awaitable[None]] | None = None,
    on_stream: Callable[[str], Awaitable[None]] | None = None,
    on_stream_end: Callable[..., Awaitable[None]] | None = None,
    pending_queue: asyncio.Queue[InboundMessage] | None = None,
    ephemeral: bool = False,
    run_extra_hooks_for_ephemeral: bool = False,
    hooks: list[AgentHook] | None = None,
    hook_factories: list[AgentTurnHookFactory] | None = None,
    tools: ToolRegistry | None = None,
    runtime: LLMRuntime | None = None,
    delivery: TurnDelivery | None = None,
    on_runtime_admitted: Callable[[LLMRuntime], Awaitable[None]] | None = None,
    attributes: Mapping[str, Any] | None = None,
    session: Session | None = None,
) -> OutboundMessage | None:
    kind = TurnKind.USER if msg.is_user_input else TurnKind.SYSTEM
    if kind is TurnKind.SYSTEM:
        destination = (
            msg.chat_id.split(":", 1) if ":" in msg.chat_id else ("cli", msg.chat_id)
        )
        key = session_key or msg.session_key_override or f"{destination[0]}:{destination[1]}"
    else:
        key = session_key or msg.session_key
    if not self.subagents.accepts_result(msg, key):
        return None
    commit_session = session is None or session.policy.persist
    if session is None:
        session = (await self.sessions.state.read(key)
                   if msg.require_existing_session or msg.session_generation is not None
                   else await self.sessions.state.get(key))
    if msg.require_existing_session and session is None:
        raise RuntimeError("required session is not active")
    if msg.session_generation is not None and (
        session is None or session.generation != msg.session_generation
    ):
        raise RuntimeError("required session generation was replaced")
    if delivery is None:
        delivery = self.turn_delivery_factory.create(msg, key)
    elif delivery.session_key != key:
        raise ValueError("turn delivery session does not match the processing session")
    t0 = time.time()
    ctx = TurnContext(
        msg=msg,
        session=session,
        session_key=key,
        turn_id=f"{key}:{time.time_ns()}",
        runtime=runtime,
        kind=kind,
        delivery=delivery,
        original_user_text=(
            None
            if kind is TurnKind.SYSTEM
            or turn_continuation.internal_continuation_inbound(msg.metadata)
            else msg.content
        ),
        turn_wall_started_at=t0,
        events=output_events(
            default=delivery.events,
            on_progress=on_progress,
            on_stream=on_stream,
            on_stream_end=on_stream_end,
        ),
        streaming=on_stream is not None or delivery.streaming,
        on_runtime_admitted=on_runtime_admitted,
        pending_queue=pending_queue,
        ephemeral=ephemeral,
        commit_session=commit_session,
        run_extra_hooks_for_ephemeral=run_extra_hooks_for_ephemeral,
        hooks=list(hooks or []),
        hook_factories=list(hook_factories or []),
        tools=tools,
        attributes=dict(attributes or {}),
    )
    # A streaming callback may be present even when the final text comes from a
    # non-streaming recovery. Only the last completed segment can suppress the
    # regular outbound message.
    if ctx.streaming:
        publish = ctx.events.publish
        segment_streamed_content = False

        async def track_output(event: AgentEvent) -> None:
            nonlocal segment_streamed_content
            if isinstance(event, StreamDeltaEvent) and event.content:
                segment_streamed_content = True
            elif isinstance(event, StreamEndEvent):
                ctx.streamed_content = segment_streamed_content
                segment_streamed_content = False
            if publish is not None:
                await publish(event)

        ctx.events = EventSink(track_output, ctx.events.accepts)

    with logger.contextualize(turn_id=ctx.turn_id, session_key=ctx.session_key):
        await self._run_turn_stage(ctx, "restore", self._restore_turn)
        await self._run_turn_stage(ctx, "compact", self._compact_session)
        if await self._run_turn_stage(ctx, "command", self._dispatch_command):
            self._log_turn_completion(ctx, outcome="command")
            return ctx.outbound
        await self._run_turn_stage(ctx, "build", self._build_turn)
        await self._run_turn_stage(ctx, "run", self._run_turn)
        await self._run_turn_stage(ctx, "save", self._persist_turn)
        await self._run_turn_stage(ctx, "respond", self._prepare_outbound)
        self._log_turn_completion(ctx)
        return ctx.outbound
```

</details>

**关键点解读：**这是阶段关系图，不是所有请求都会走完的强制调用栈。例如命令可能提前结束。先用这条线定位某个功能归谁管，再进入具体方法；不要在多个章节重复粘贴整段单轮编排代码。

### 片段 2：MCP 工具注册流程

<a id="method-mcpprovider-connect"></a>

源码：[nanobot/agent/tools/mcp.py:1425](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/mcp.py#L1425)，`MCPProvider.connect` 完整方法。

```python
# 筛选尚未连接且授权条件满足的服务，管理连接状态和关闭期间的资源回收。
async def connect(self) -> None:
    """Connect configured servers that are not currently live."""
    async with self._lock:
        if self._closing:
            return
        # 只选择已配置但当前尚未连接的服务。
        configured_missing = {
            name: cfg
            for name, cfg in self._servers.items()
            if name not in self._connections
        }
        oauth_servers = {
            name: cfg
            for name, cfg in configured_missing.items()
            if cfg.auth == "oauth"
        }
        authorization_pending: set[str] = set()
        if oauth_servers:
            from nanobot.agent.tools.mcp_oauth import mcp_oauth_has_credentials

            authorization_pending = {
                name
                for name, cfg in oauth_servers.items()
                if not mcp_oauth_has_credentials(name, cfg.url)
            }
        for name in authorization_pending:
            self._runtime_statuses.pop(name, None)
        # 排除尚待授权的服务，避免无效连接尝试。
        missing_servers = {
            name: cfg
            for name, cfg in configured_missing.items()
            if name not in authorization_pending
        }
        if not missing_servers:
            return
        self._set_runtime_status(missing_servers, "connecting")
        try:
            # 建立服务连接并把发现的工具注册进统一工具表。
            connected = await connect_mcp_servers(missing_servers, self._registry, network=self._network)
            if self._closing:
                await _close_mcp_connections(connected)
                return
            # 接管新连接，由 Provider 管理后续恢复和关闭。
            self._connections.update(connected)
            self._record_connection_result(missing_servers, connected)
            self._attach_reconnect_handlers(connected)
            if connected:
                logger.info("MCP connected servers: {}", sorted(connected))
            else:
                logger.warning(
                    "No MCP servers connected successfully "
                    "(will retry on the next readiness check)"
                )
        # 单独处理取消信号，避免把真实取消误当成普通运行错误。
        except asyncio.CancelledError:
            self._set_runtime_status(missing_servers, "failed")
            if task_is_cancelling():
                raise
            logger.warning(
                "MCP connection cancelled (will retry on the next readiness check)"
            )
        # 处理 BaseException，按本方法约定记录、转换或继续传播错误。
        except BaseException as exc:
            self._set_runtime_status(missing_servers, "failed")
            logger.warning(
                "Failed to connect MCP servers "
                "(will retry on the next readiness check): {}",
                exc,
            )
```

**关键点解读：**发现阶段让模型知道“可以做什么”，调用阶段才真的执行。服务名、远端工具名与本地注册名需要在调试时对应起来；不要把连接成功当作每个工具调用都已成功。

### 片段 3：Provider 注册表与工厂

源码：[nanobot/providers/routing.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/providers/routing.py#L1)。

<details>
<summary>展开 _make_provider_core() 完整方法（102 行源码）</summary>

<a id="method-make-provider-core"></a>

源码：[nanobot/providers/factory.py:55](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/providers/factory.py#L55)，`_make_provider_core` 完整方法。

```python
# 解析供应商路由，根据后端类型创建适配器，再注入生成参数。
def _make_provider_core(
    config: Config,
    *,
    preset: ModelPresetConfig,
    model: str | None = None,
) -> LLMProvider:
    # 先解析供应商与后端，再进入对应适配器的构造分支。
    setup = resolve_provider_route(
        config,
        preset=preset,
        model=model,
    )
    model = setup.model
    provider_name = setup.provider_name
    p = setup.provider_config
    spec = setup.spec
    backend = setup.backend
    proxy = config.network.resolve_proxy(p.use_proxy) if p else None

    if backend == "openai_codex":
        from nanobot.providers.openai_codex_provider import OpenAICodexProvider

        provider = OpenAICodexProvider(
            default_model=model,
            proxy=proxy,
            extra_body=p.extra_body if p else None,
            provider_name=provider_name,
        )
    elif backend == "xai_grok":
        from nanobot.providers.xai_grok_provider import XAIGrokProvider

        provider = XAIGrokProvider(
            default_model=model,
            proxy=proxy,
            extra_body=p.extra_body if p else None,
            provider_name=provider_name,
        )
    elif backend == "azure_openai":
        from nanobot.providers.azure_openai_provider import AzureOpenAIProvider

        if p is None or p.api_base is None:
            raise RuntimeError("validated Azure provider setup is missing api_base")
        provider = AzureOpenAIProvider(
            api_key=p.api_key or "",
            api_base=p.api_base,
            default_model=model,
            provider_name=provider_name,
        )
    elif backend == "github_copilot":
        from nanobot.providers.github_copilot_provider import GitHubCopilotProvider

        provider = GitHubCopilotProvider(
            default_model=model, provider_name=provider_name,
            model_api=setup.model_api,
        )
    elif backend == "anthropic":
        from nanobot.providers.anthropic_provider import AnthropicProvider

        custom_spec = spec if spec and spec.backend == "openai_compat" else None
        api_key = p.api_key if p else None
        if custom_spec is not None:
            api_key = api_key or "no-key"
        provider = AnthropicProvider(
            api_key=api_key,
            api_base=config.get_api_base(model, preset=preset),
            default_model=model,
            extra_headers=_provider_extra_headers(spec, p),
            extra_body=p.extra_body if p else None,
            extra_query=p.extra_query if p else None,
            proxy=proxy,
            spec=custom_spec,
            provider_name=provider_name,
        )
    elif backend == "bedrock":
        from nanobot.providers.bedrock_provider import BedrockProvider

        provider = BedrockProvider(
            api_key=p.api_key if p else None,
            api_base=p.api_base if p else None,
            default_model=model,
            region=getattr(p, "region", None) if p else None,
            profile=getattr(p, "profile", None) if p else None,
            extra_body=p.extra_body if p else None,
            provider_name=provider_name,
        )
    else:
        from nanobot.providers.openai_compat_provider import OpenAICompatProvider

        provider = OpenAICompatProvider(
            api_key=p.api_key if p else None,
            api_base=config.get_api_base(model, preset=preset),
            default_model=model,
            extra_headers=_provider_extra_headers(spec, p),
            spec=spec,
            extra_body=p.extra_body if p else None,
            model_api=setup.model_api,
            extra_query=p.extra_query if p else None,
            proxy=proxy,
            provider_name=provider_name,
        )

    # 把模型预设中的生成参数注入刚创建的适配器。
    provider.generation = preset.to_generation_settings()
    return provider
```

</details>

**关键点解读：**“供应商”是配置与路由概念，“适配器”是实现。兼容 OpenAI 协议的多个服务可能共享适配器，因此不需要每出现一个服务就寻找同名 Python 文件。

<a id="section-4-12"></a>

## 4.12 面试高频题

### Q1: 请描述 AgentLoop 的执行流程

从 MessageBus 等待消息，经分发进入会话协调，再准备上下文、调用复用的 Runner、保存会话并安排交付。重点说明接收循环与单轮执行的分工，以及控制消息为什么仍可被处理。

### Q2: save_memory 虚拟工具是怎么工作的？

这道旧题的前提不再适用于当前 Runner。应先指出版本差异，再说明当前 Registry 的工具调用链、MemoryStore 的读写，以及会话保存和记忆整理的区别。不能为了迎合题目而复述不存在的拦截分支。

### Q3: 为什么工具结果要截断到 16000 字符？

当前 `maxToolResultChars` 默认值为 16000，限制模型可见工具结果有助于控制上下文和成本。它是可配置预算，不等于 `_save_turn()` 把最终回复截到这个长度。过度截断可能丢失关键信息，应结合工具输出结构与后续读取策略理解。

### Q4: 如何新增一个 LLM 供应商（如文心一言）？

先判断现有兼容适配器是否覆盖协议；再核对配置、路由和工厂，必要时实现 `LLMProvider` 适配。最后验证请求、工具调用、流式输出和错误语义。只往注册表添加名称不代表完整接入。

<a id="section-4-13"></a>

## 4.13 本章总结

### 源码阅读建议

1. 先读 `AgentLoop.run()`，画出等待与分发关系，理解同一会话怎样协调输入。
2. 再读 `AgentRunner.run()` 与 `_run_core()`，跟一次工具调用前后的消息列表变化。
3. 回到 ContextBuilder，解释这次请求具体包含哪些输入。
4. 查看 `_save_turn()` 与 SessionState，区分会话历史、长期记忆和整理后的摘要。
5. 最后沿工具注册、MCP、渠道和 Provider 向外扩展，每次只追一条调用链。

读完的判断标准：不用看文档，也能解释“输入在哪里等待、工具是谁执行、上下文从哪里来、历史由谁提交”。遇到不熟悉的分支，再从本章的源码链接进入完整实现。

## 下一章

继续阅读 [05 - MCP 协议](../05-mcp-protocol/README.md)，理解工具能力如何跨进程接入。
