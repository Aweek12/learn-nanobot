# 07 - 会话、长期记忆与压缩

> 已按本地 nanobot `0.4.1`、提交 `27f1a836b5c04a30763dd91960059f81c05bf505` 核对。真实源码、API 签名、伪代码与配置示例分别标注。详见 [版本核对记录](../LOCAL_SOURCE_BASELINE.md)。

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

源码：[nanobot/agent/memory.py:66](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L66)，`MemoryStore.__init__`。

```python
self.memory_dir = ensure_dir(workspace / "memory")
self.memory_file = self.memory_dir / "MEMORY.md"
self.history_file = self.memory_dir / "history.jsonl"
self.legacy_history_file = self.memory_dir / "HISTORY.md"
```

**关键点解读：**

1. `MEMORY.md` 存放整理后的长期知识，`history.jsonl` 存放逐条历史记录，读取用途不同。
2. `legacy_history_file` 是旧历史文件的位置，构造阶段还有迁移处理，并不是继续把新历史写回旧文件。
3. 本轮会话消息由 SessionState 与会话存储管理；这里的文件路径不能代替会话数据库的位置。

源码：[nanobot/agent/memory.py:269](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L269)。

流程伪代码：

```text
append_history(entry, max_chars, session_key):
    清理条目内容，并应用长度预算
    在写入锁内分配新 cursor
    将 cursor、timestamp、content 等写成一行 JSON
    更新游标文件，返回 cursor
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

源码：[nanobot/agent/memory.py:1065](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L1065)。

流程伪代码：

```text
已接受的 transcript + previous_summary
    → Consolidator.summarize_transcript(...)
    → 使用 runtime 提供的模型能力整理摘要
    → 返回摘要文本或 None，并按 persist 条件处理持久化
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
