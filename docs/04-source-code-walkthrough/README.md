# 04 - 源码走读：当前核心实现

> 已按本地 nanobot `0.4.1`、提交 `27f1a836b5c04a30763dd91960059f81c05bf505` 核对。真实源码、API 签名、伪代码与配置示例分别标注。详见 [版本核对记录](../LOCAL_SOURCE_BASELINE.md)。

代码依据本地固定提交。完整方法与仅保留签名明确区分；长流程用伪代码表达，不将示意变量冒充真实 API。

## 目录

- [4.1 源码目录与入口](#section-4-1)
- [4.2 AgentLoop 的执行阶段](#section-4-2)
- [4.3 _save_turn：真实历史保存](#section-4-3)
- [4.4 AgentRunner：复用对象与单次 run](#section-4-4)
- [4.5 ContextBuilder：上下文来源](#section-4-5)
- [4.6 MemoryStore、Consolidator 与预算](#section-4-6)
- [4.7 ToolRegistry 与执行](#section-4-7)
- [4.8 Hook 与事件接口](#section-4-8)
- [4.9 MCP 与子任务入口](#section-4-9)
- [4.10 消息、渠道与 Provider](#section-4-10)
- [4.11 配置与阅读自查](#section-4-11)

<a id="section-4-1"></a>

## 4.1 源码目录与入口

```text
nanobot/
├── cli/、gateway/            命令与共享后端
├── agent/loop.py             轮次编排
├── agent/runner.py           模型工具循环
├── agent/context.py          上下文
├── agent/tools/              工具协议与实现
├── providers/                路由、工厂、适配器
├── session/                  会话 owner 与 SQLite
├── bus/、events.py           消息与事件
├── channels/<channel>/       自包含渠道包
├── security/                 访问与隔离
└── skills/、templates/       说明与提示词资源
```

不要套用旧 cli.py、bus/message_bus.py、tools/spawn.py、providers/ollama.py 等布局。模型服务名不必对应同名文件，要跟真实路由。


<a id="section-4-2"></a>

## 4.2 AgentLoop 的执行阶段

_process_message 经过 restore、compact、command、build、run、save、respond；命令可提前完成。_run_turn 把本轮输入交给 _run_agent_loop，后者构造 AgentRunSpec 调用已有 Runner。

**真实源码节选**：[nanobot/agent/loop.py:2296](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/loop.py#L2296)，`AgentLoop._run_turn`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
async def _run_turn(self, ctx: TurnContext) -> None:
    runtime = ctx.require_runtime()
    ctx.run_started_at = time.time()
    await ctx.delivery.running(started_at=ctx.run_started_at)
    assert ctx.transcript_input is not None
    with capture_message_deliveries() as message_sends:
        result = await self._run_agent_loop(
            ctx.transcript_input,
            runtime=runtime,
            streaming=ctx.streaming,
            session=ctx.session,
            pending_queue=ctx.pending_queue,
            ephemeral=ctx.ephemeral,
            run_extra_hooks_for_ephemeral=ctx.run_extra_hooks_for_ephemeral,
            hooks=ctx.hooks,
            hook_factories=ctx.hook_factories,
            tools=ctx.tools,
            request_context=ctx.request_context,
            provider_state=ctx.provider_state,
            initial_message=ctx.msg,
            events=ctx.events,
            commit_session=ctx.commit_session,
        )
    ctx.final_content = result.final_content
    ctx.all_messages = result.messages
    ctx.summary_checkpoint = result.summary_checkpoint
    ctx.provider_compaction_applied = result.provider_compaction_applied
    ctx.stop_reason = result.stop_reason
    ctx.failure_error_kind = result.failure_error_kind
    if (
        ctx.kind is TurnKind.USER
        and (ctx.delivery.route.channel, ctx.delivery.route.chat_id) in message_sends
        and (not result.had_injections or result.stop_reason == "empty_final_response")
    ):
        ctx.suppress_response = True
    ctx.usage = result.usage
    ctx.delivery.record_usage(result.round_usages)
```


<a id="section-4-3"></a>

## 4.3 _save_turn：真实历史保存

当前接收 Session、messages、skip 和检查点等参数，不截取用户前200字、回复前500字写入 HISTORY.md。messages[skip:] 跳过消息条目；persisted_message 整理元数据，工具结果需有合法调用 ID，媒体与空消息有清理规则。

**真实源码节选**：[nanobot/agent/loop.py:2464](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/loop.py#L2464)，`AgentLoop._save_turn`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
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
        session.commit_summary_checkpoint(
            summary_checkpoint.summary,
            insert_at=insert_at,
        )

    for message_index, message in enumerate(messages[skip:], start=skip):
        # Insert against the raw transcript index before filtering the
        # message so persistence cleanup cannot shift the H/Δ boundary.
        if summary_checkpoint is not None and checkpoint_boundary == message_index:
            session.commit_summary_checkpoint(summary_checkpoint.summary)

        entry = persisted_message(message)
        followup_id = entry.pop(PENDING_FOLLOWUP_ID_KEY, None)
        role, content = entry.get("role"), entry.get("content")
        if role == "assistant" and not content and not entry.get("tool_calls"):
            continue  # skip empty assistant messages — they poison session context
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
        entry.setdefault("timestamp", datetime.now().isoformat())
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
        session.commit_summary_checkpoint(summary_checkpoint.summary)
    if turn_latency_ms is not None and last_assistant_idx is not None:
        session.messages[last_assistant_idx]["latency_ms"] = int(turn_latency_ms)
    session.consumed_followup_ids |= saved_followup_ids
    session.updated_at = datetime.now()
```

外层 _persist_turn 在需要时调用 sessions.state.finish_turn(session)。日志中的80/120字符是预览，不能当作历史正文策略；这里也不承诺所有媒体字节原样保存。


<a id="section-4-4"></a>

## 4.4 AgentRunner：复用对象与单次 run

初始化共享成员与本次 messages、usage、停止原因要分开。输入 AgentRunSpec 带 runtime、tools、预算、transcript、Hook 和回调；结果 AgentRunResult 返回内容、消息、用量和停止原因。

**真实源码节选**：[nanobot/agent/runner.py:155](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/runner.py#L155)，`AgentRunner.__init__`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
def __init__(self) -> None:
    self.context_governor = ContextGovernor()
```

**真实源码节选**：[nanobot/agent/runner.py:284](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/runner.py#L284)，`AgentRunner.run`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
async def run(self, spec: AgentRunSpec) -> AgentRunResult:
    hook = spec.hook or AgentHook()
    has_run_hooks = hook.has_run_hooks()
    messages: list[dict[str, Any]] = []
    context = AgentRunHookContext(messages=[])
    llm_usage_source_token = bind_llm_usage_source(
        spec.llm_usage_source or source_from_session_key(spec.session_key)
    )

    try:
        try:
            try:
                messages, compaction = await self.prepare_prompt(self._initial_transcript_and_compaction, spec)
                context.messages = await self._run_context_snapshot(hook, messages) if has_run_hooks else []
                await hook.before_run(context)
                result = await self._run_core(spec, hook, messages, compaction)
            except Exception as exc:
                context.messages = await self._run_context_snapshot(hook, messages) if has_run_hooks else []
                context.stop_reason = "error"
                context.error = f"Error: {type(exc).__name__}: {exc}"
                context.exception = exc
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
                    await hook.on_error(context)
                await hook.after_run(context)
                return result
        except asyncio.CancelledError as exc:
            context.stop_reason = "cancelled"
            context.error = None
            context.exception = exc
            raise
        finally:
            if has_run_hooks:
                try:
                    if isinstance(context.exception, asyncio.CancelledError):
                        context.messages = await self._run_context_snapshot(hook, messages)
                    else:
                        context.messages = await self._run_context_snapshot(hook, messages)
                except asyncio.CancelledError as exc:
                    context.stop_reason = "cancelled"
                    context.error = None
                    context.exception = exc
                    context.messages = await self._run_context_snapshot(hook, messages)
                    raise
    finally:
        try:
            if context.exception is None:
                await hook.on_finally(context)
            else:
                try:
                    await hook.on_finally(context)
                except Exception:
                    logger.opt(exception=tool_log_content_allowed()).error(
                        "AgentHook.on_finally error after {}",
                        context.stop_reason or "run exception",
                    )
        finally:
            reset_llm_usage_source(llm_usage_source_token)
```

**伪代码**：只表达流程，操作名不是可直接调用的 nanobot API。

```text
准备本次 transcript 与压缩状态
在迭代/已接纳继续条件下：
  处理输入注入并构造请求
  调用实际 Provider，累计响应与用量
  如果 should_execute_tools：保存声明，执行并回传配对结果
  否则处理最终内容、错误、空响应和长度恢复
返回结构化结果，保证异常/取消收尾
```

AgentDefaults.max_tool_iterations 默认200，子管理器默认也读取配置，不是主40/子15。Goal、followup、重试各有边界，不能把每次网络请求等同一次工具迭代。


<a id="section-4-5"></a>

## 4.5 ContextBuilder：上下文来源

**真实源码节选**：[nanobot/agent/context.py:87](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/context.py#L87)，`ContextBuilder.build_system_prompt`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
def build_system_prompt(
    self,
    *,
    channel: str | None = None,
    session_summary: SessionSummary | None = None,
    workspace: Path | None = None,
    include_memory: bool = True,
) -> str:
    """Build the system prompt from identity, bootstrap files, memory, and skills."""
    root = workspace or self.workspace
    parts = [self._get_identity(channel=channel, workspace=root)]

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

    if include_memory:
        memory = self.memory.read_memory()
        if memory and not self._is_template_content(memory, "memory/MEMORY.md"):
            parts.append(memory)
            record_source(self.memory.memory_file, memory)
        elif self.memory.memory_file.is_file():
            record_source(self.memory.memory_file, "", loaded=False)

    with self.skills.snapshot():
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

身份、bootstrap、工具契约、项目、记忆、always技能、其他技能摘要和会话摘要有各自条件。include_memory 和模板/空内容检查影响注入，不能写无条件始终全量注入。TranscriptInput/build_transcript 连接结构化历史与当前输入。


<a id="section-4-6"></a>

## 4.6 MemoryStore、Consolidator 与预算

MemoryStore 负责文件，MemoryArchiver 归档来源，Consolidator 协调摘要，ContextGovernor 管请求预算。Dream 整理 MEMORY/SOUL/USER；不存在“所有普通工具调用都由 Runner 拦截 save_memory”的当前主路径。

**真实源码节选**：[nanobot/agent/memory.py:221](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L221)，`MemoryStore.read_memory`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
def read_memory(self) -> str:
    return self.read_file(self.memory_file)
```

**真实源码节选**：[nanobot/agent/memory.py:224](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L224)，`MemoryStore.write_memory`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
def write_memory(self, content: str) -> None:
    self.memory_file.write_text(content, encoding="utf-8")
```

超预算会区分已接收历史和新工具文本，必要时返回要求缩小读取范围的错误，保持回放一致，不是简单统一截取前16000字符。


<a id="section-4-7"></a>

## 4.7 ToolRegistry 与执行

Tool 公开 name/description/parameters/execute 和配置/并发属性。注册 register(tool)，导出 get_definitions；不是 get_tool_schemas 或 register(name, handler)。

**真实源码节选**：[nanobot/agent/tools/registry.py:120](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/registry.py#L120)，`ToolRegistry.prepare_call`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
def prepare_call(
    self,
    name: str,
    params: Any,
) -> tuple[Tool | None, Any, str | None]:
    """Resolve, cast, and validate one tool call."""
    tool = self.get(name)
    if not tool:
        suggestion = self._suggest_name(str(name))
        hint = f" Did you mean '{suggestion}'? Tool names must match exactly." if suggestion else ""
        return None, params, (
            ToolResult.error(
                f"Error: Tool '{name}' not found.{hint} Available: {', '.join(self.tool_names)}"
            )
        )
    # Compatibility for external tools that still implement the legacy
    # setter protocol. Built-ins read the authoritative ContextVar
    # directly and never copy routing state.
    if isinstance(tool, ContextAware) and (ctx := current_request_context()) is not None:
        tool.set_context(ctx)

    params = self._coerce_params(tool, params)
    if not isinstance(params, dict):
        return tool, params, (
            ToolResult.error(
                f"Error: Tool '{name}' parameters must be a JSON object, got "
                f"{type(params).__name__}. Use named parameters like "
                'tool_name(param1="value1", param2="value2") matching the tool schema.'
            )
        )

    cast_params = tool.cast_params(cast(dict[str, Any], params))
    errors = tool.validate_params(cast_params)
    if errors:
        return tool, cast_params, (
            ToolResult.error(f"Error: Invalid parameters for tool '{name}': " + "; ".join(errors))
        )
    return tool, cast_params, None
```

JSON 解码按 Schema 指导，允许字符串的联合类型等不随意改写。execution.py 组织工具任务和结果收敛；ToolResult.is_error 不等于任意输出以 Error 开头。


<a id="section-4-8"></a>

## 4.8 Hook 与事件接口

AgentHook 有运行、模型、迭代、流式和工具生命周期。不同上下文不是同一份全局状态。CompositeHook 通常隔离异步异常，但 _reraise 与 finalize_content 的行为不同。

**API 签名节选，省略实现**：[nanobot/agent/hook.py:126](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/hook.py#L126)。

```python
async def before_execute_tool(
    self,
    context: AgentHookContext,
    tool_call: ToolCallRequest,
    tool: Any,
    params: Any,
) -> None:
    ...
```

AgentProgressHook 转为进度，SDKCaptureHook 捕获结果，turn_hooks 组装本轮 Hook。回调接口、事件值和 EventSink/MessageBus 不同，不能混称为一个消息队列。


<a id="section-4-9"></a>

## 4.9 MCP 与子任务入口

**真实源码节选**：[nanobot/agent/tools/mcp.py:743](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/mcp.py#L743)，`MCPToolWrapper.execute`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
async def execute(self, **kwargs: Any) -> str:
    result = await self._request_with_recovery(
        lambda session: session.call_tool(self._original_name, arguments=kwargs),
        self._tool_timeout,
        "tool",
    )
    if isinstance(result, ToolResult):
        return result
    try:
        rendered = self._render_call_result(result.content, kwargs)
        if getattr(result, "isError", False):
            return ToolResult.error(rendered)
        return rendered
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

Wrapper 转发到外部 ClientSession，并处理结构化错误与返回渲染。MCPProvider 管连接。SubagentTool 的 name 是 subagent，action 为 create/send/cancel/check，而非当前仍暴露 SpawnTool；管理器有独立 Runner 与任务归属检查。


<a id="section-4-10"></a>

## 4.10 消息、渠道与 Provider

InboundMessage 正文为 content，session_key 是属性并可被 override；OutboundMessage 还有 event/delivery 等字段。BaseChannel 的基本生命周期是 start/stop/send，而非统一 receive() 轮询。

Provider 使用 LLMProvider 共同契约，经 registry、routing、factory 到兼容或专用实现。不能把服务清单硬映射为 openai.py、deepseek.py 等文件。

[nanobot/bus/events.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/bus/events.py) · [nanobot/providers/routing.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/providers/routing.py)


<a id="section-4-11"></a>

## 4.11 配置与阅读自查

config/schema.py 使用 Pydantic，loader.py 提供 load_config/save_config 等函数。环境引用 ${VAR} 不是 shell 默认值语法，缺失时报明确配置错误。工具与渠道拥有相邻配置，根模型组装结构。

阅读顺序：模型/工具契约 → Runner正常分支 → Context → Loop阶段 → SessionState → 安全与取消。用 test_runner_core、test_tool_registry、test_context_builder、test_session_store、test_task_cancel 核对。自查对象创建、模型迭代、工具副作用、SQLite提交和出站显示各发生在哪里。

## 下一章

[继续第 05 章](../05-mcp-protocol/README.md)
