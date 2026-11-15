# 07 - 会话、长期记忆与压缩

> 已按本地 nanobot `0.4.1`、提交 `27f1a836b5c04a30763dd91960059f81c05bf505` 核对。方法使用本地新版完整源码并配中文 # 注释解读，配置示例单独标注。详见 [版本核对记录](../LOCAL_SOURCE_BASELINE.md)。

原“两份Markdown文件管理全部记忆”的解释已替换为当前的 SQLite 会话、整理来源、摘要治理和 Dream 分工。

## 目录

- [7.1 三种历史与当前布局](#section-7-1)
- [7.2 MemoryStore 接口](#section-7-2)
- [7.3 会话 owner 与摘要](#section-7-3)
- [7.4 Consolidator 与 ContextGovernor](#section-7-4)
- [7.5 Dream 与注入条件](#section-7-5)
- [7.6 练习与面试](#section-7-6)

<a id="section-7-1"></a>

## 7.1 三种历史与当前布局

预算内模型上下文、SQLite会话和记忆整理 journal 不同。

```text
配置目录/sessions/<workspace-id>/sessions.sqlite3
Agent工作区/SOUL.md
Agent工作区/USER.md
Agent工作区/memory/MEMORY.md
Agent工作区/memory/history.jsonl
Agent工作区/memory/.cursor
Agent工作区/memory/.dream_cursor
```

HISTORY.md 仅作为旧记忆格式迁移来源；旧会话JSONL也有导入路径，不表示当前主存储还是JSONL。项目选择通常不搬迁Agent资料。


<a id="section-7-2"></a>

## 7.2 MemoryStore 接口

<a id="method-memorystore-init"></a>

源码：[nanobot/agent/memory.py:66](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L66)，`MemoryStore.__init__` 完整方法。

```python
# 确定记忆与历史路径，初始化游标、写锁和版本存储，并迁移旧历史。
def __init__(self, workspace: Path, max_history_entries: int = _DEFAULT_MAX_HISTORY):
    self.workspace = workspace
    self.max_history_entries = max_history_entries
    self.memory_dir = ensure_dir(workspace / "memory")
    self.memory_file = self.memory_dir / "MEMORY.md"
    self.history_file = self.memory_dir / "history.jsonl"
    self.legacy_history_file = self.memory_dir / "HISTORY.md"
    self.soul_file = workspace / "SOUL.md"
    self.user_file = workspace / "USER.md"
    self._cursor_file = self.memory_dir / ".cursor"
    self._dream_cursor_file = self.memory_dir / ".dream_cursor"
    self._corruption_logged = False
    self._malformed_entry_logged = False
    self._oversize_logged = False
    self._dream_prompt_oversize_logged = False
    self._append_lock = threading.Lock()
    self._git = GitStore(workspace, tracked_files=[
        "SOUL.md", "USER.md", "memory/MEMORY.md", "memory/.dream_cursor",
    ])
    self._maybe_migrate_legacy_history()
```

**关键点解读：**

1. `MEMORY.md` 存放整理后的长期知识，`history.jsonl` 存放逐条历史记录，读取用途不同。
2. `legacy_history_file` 是旧历史文件的位置，构造阶段还有迁移处理，并不是继续把新历史写回旧文件。
3. 本轮会话消息由 SessionState 与会话存储管理；这里的文件路径不能代替会话数据库的位置。

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

**关键点解读：**

游标和追加在同一把锁保护下，避免并发写入分配重复位置。`max_chars` 是历史条目的预算，既不是会话消息数量，也不是旧例子中的用户 200 字、回复 500 字规则。历史整理通过游标知道已经处理到哪里。

append_history 接收字符串并同步返回cursor，不是 await 写 turn_summary 字典。write_memory、write_soul、write_user 有各自文件语义，不代替会话提交。


<a id="section-7-3"></a>

## 7.3 会话 owner 与摘要

SessionState 管运行时修改，SqliteSessionStore 管事务，模型消息与展示事实分开。_save_turn 整理新增条目、工具配对与摘要边界，外层 finish_turn 完成提交。generation 防止迟到写入恢复已删除或重置会话；fork 维护相应前缀。

存储完整历史不等于每轮发送全部历史；实际回放受摘要与上下文预算影响。

[nanobot/session/state.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/session/state.py)


<a id="section-7-4"></a>

## 7.4 Consolidator 与 ContextGovernor

当前类为 Consolidator、MemoryArchiver，不是原教程的 MemoryConsolidator。请求治理区分模型已接收历史与新工具结果，必要时摘要或返回更小读取范围的预算错误。不是固定超过50条记录便写两份Markdown。

<a id="method-consolidator-summarize-transcript"></a>

源码：[nanobot/agent/memory.py:1065](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L1065)，`Consolidator.summarize_transcript` 完整方法。

```python
# 选择可整理消息，计算摘要预算，并交给 archiver 生成受预算限制的摘要。
async def summarize_transcript(
    self,
    accepted_messages: list[dict[str, Any]],
    previous_summary: str | None,
    *,
    runtime: LLMRuntime,
    session_key: str,
    tools: list[dict[str, Any]],
    provider_state: ProviderConversationState | None = None,
    persist: bool = True,
) -> str | None:
    """Summarize the exact transcript prefix already accepted by the model."""
    source_messages = [
        dict(message)
        for message in accepted_messages
        if message.get("role") != "system"
    ]
    if not source_messages:
        return None

    max_output_tokens = max(0, runtime.generation.max_tokens)
    input_token_budget = runtime.context_window_tokens - max_output_tokens
    checkpoint_tokens = min(
        max_output_tokens,
        max(1, (input_token_budget - self._SAFETY_BUFFER) // 2),
    )

    # 把选定消息和预算交给摘要归档入口。
    summary = await self.archiver.archive(
        source_messages,
        runtime=runtime,
        session_key=session_key,
        history=accepted_messages,
        request_tools=tools,
        previous_summary=previous_summary,
        input_token_budget=input_token_budget,
        fallback_max_tokens=max(1, checkpoint_tokens),
        provider_state=provider_state,
        persist=persist,
    )
    if summary is None:
        return None
    # 即使摘要生成成功，也要满足输出 token 预算。
    return truncate_text_to_tokens(summary, max(1, max_output_tokens))
```

**关键点解读：**

摘要是在既有总结基础上整理新内容。`runtime` 决定模型执行条件，`session_key` 标识来源，`persist` 控制相应保存行为。返回值允许为空，因此调用方不能假定每次请求都会得到新摘要。理解这一接口后，再读内部请求和预算处理。


<a id="section-7-5"></a>

## 7.5 Dream 与注入条件

Dream 用未处理来源整理 MEMORY、SOUL、USER，并维护进度，可能判定不需要新增长期信息。是否启用、周期和模型来自 DreamConfig 等实际配置。

ContextBuilder 按 include_memory、模板和空值等条件加载记忆，always技能和摘要另行处理。不能写“每个文件始终全量放进system”。

[docs/memory.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/memory.md)


<a id="section-7-6"></a>

## 7.6 练习与面试

先读 test_session_store 的完整历史和元数据往返，再读 test_consolidator/test_context_governance。真实偏好实验先确认写入，再用同实例新会话核对来源。口头说记住、同会话回答正确、数据库可检索、长期资料已加载是不同证据。

能解释日志预览与正文、摘要与记忆、持久化与当次上下文的差异。不要编辑运行中的数据库来制造测试结果。

## 下一章

[继续第 08 章](../08-skills-and-tools/README.md)
