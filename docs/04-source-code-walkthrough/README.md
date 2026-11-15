# 04 - 源码逐行解读

本章保留原教程的目录总览与讲解主题，按本地 nanobot `0.4.1`（`27f1a836b5c04a30763dd91960059f81c05bf505`）核对实现。每节先说明问题，再展示核心代码或流程伪代码，最后解释关键点；完整实现从源码链接继续阅读。

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

原教程用“每收到一条消息创建一个 Runner”解释 Loop，新版已经不是这个关系。先抓住构造阶段这一行：

```python
self.runner = AgentRunner()
```

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

原教程这个小节仍然讲 `async def run(self)`。新版保留“等待输入，再分发”的主线，核心等待代码是：

源码：[nanobot/agent/loop.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/loop.py#L1408)，`AgentLoop.run`。

```python
async with asyncio.timeout(1.0):
    msg = await self.bus.consume_inbound()
```

把超时、分发与关闭放回一起，可以这样读：

流程伪代码：

```text
run():
    标记循环运行中
    while 仍在运行:
        在 1 秒超时范围内等待 consume_inbound()
        若超时：按需安排过期会话检查，继续等待
        在 admission 活动范围内分发消息 _dispatch_inbound(msg)
    finally: aclose() 释放运行时资源
```

**关键点解读：**

1. `await` 等待队列时让出事件循环，不会让整个进程停止接收其他异步事件。
2. 1 秒是本次取消息的等待窗口，不是模型回答的时间限制，也不是每秒只能处理一条消息。
3. `_dispatch_inbound()` 识别优先命令并路由到会话队列。因此 `/stop` 能走控制路径；不能直接把分发理解为等待一整轮模型回答结束。
4. 同一会话已有活动队列时，新输入进入对应的协调路径；不同会话可以独立推进，是否再受全局闸门限制取决于配置。
5. `finally` 保证退出时清理资源。真实取消要传播出去，这与普通取消息失败后继续等待不同。

**跟一次输入：**用户 A 发“读 README”后，模型可能还在调用工具；此时 A 发 `/stop`，主消费循环仍有机会取到控制消息。若把完整 ReAct 循环直接堵在消费循环中，这种响应性就难以保证。

### _register_default_tools() 方法

保留原教程“默认工具如何注册”的问题，但新版应转到 `ToolLoader.load()`，不要再寻找旧示例中的同名方法。

源码：[nanobot/agent/tools/loader.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/loader.py#L89)，`ToolLoader.load`。

流程伪代码：

```text
发现内置工具类和插件工具类
按 scope、启用配置和运行条件筛选
创建工具实例，并处理调用方已经提供的工具
注册到 ToolRegistry，供模型取得定义并按名称调用
```

**关键点解读：**发现解决“有哪些工具类”，创建解决“依赖怎么传进去”，注册解决“用什么名字找到实例”。拆开这三步，才能理解为什么新增工具通常无需在 Loop 中继续堆注册语句。具体可用工具还取决于配置，目录中存在文件不等于本轮一定启用。

### _save_turn() 方法

原示例把用户前 200 字和回复前 500 字存到 `HISTORY.md`，这不再是当前方法的实现。新版这里处理的是本轮结构化会话增量，而且方法是同步 `def`。

源码：[nanobot/agent/loop.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/loop.py#L2464)，`AgentLoop._save_turn`。

流程伪代码：

```text
_save_turn(session, messages, skip, ...):
    校验摘要检查点位置
    遍历 messages[skip:] 的新增消息
        转为持久化消息格式
        跳过空 assistant，校验 tool_call_id 的配对和重复结果
        清理不能直接持久化的媒体块，补充时间戳
        追加到 session.messages，并处理摘要检查点
    记录本轮延迟、后续输入等状态
```

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

```python
hook = spec.hook or AgentHook()
has_run_hooks = hook.has_run_hooks()
messages: list[dict[str, Any]] = []
context = AgentRunHookContext(messages=[])
```

**关键点解读：**`messages` 和 `context` 是这次调用的局部变量；`spec.hook` 可以提供本轮扩展行为，缺省时使用基础 Hook。不要看到共享 Runner 就认定不同会话会共用同一个局部列表，也不要据此忽略工具实例和会话层自己的并发约束。

### run() 方法：ReAct 循环核心

新版把运行生命周期留在 `run()`，把迭代主体拆到 `_run_core()`。读原来的 run 主题时，需要顺着这个调用走一步。

源码：[nanobot/agent/runner.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/runner.py#L384)，`AgentRunner._run_core`。

流程伪代码：

```text
run(spec):
    准备初始 transcript 和压缩状态
    before_run → _run_core → after_run
    出错走 on_error，退出时走 on_finally

_run_core(...):
    在预算和继续条件允许时循环
        合并待处理输入，准备本次模型上下文
        before_iteration → 请求模型
        若 should_execute_tools：写入 assistant 工具调用并执行工具
            追加工具结果，继续下一次模型请求
        否则处理最终输出；结合后续输入和继续策略决定结束
    返回消息、最终文本、用量与停止原因
```

**关键点解读：**

1. ReAct 在这里体现为“模型选择行动 → 工具返回观察 → 模型继续判断”，不是固定交替输出某种思维文本。
2. 工具调用先以 assistant 消息记录，再追加带调用 ID 的结果，下一次模型请求才能知道工具查到了什么。
3. 没有工具调用时通常进入最终输出路径，但新版还会处理输入注入、继续和收尾策略，不能用一个无条件 `break` 概括所有分支。
4. 预算限制用于防止无休止循环；停止原因比“返回一段字符串”更丰富，调用方还需要区分正常完成、取消和错误。

**例子：**用户问“README 的安装步骤是什么”。模型先调用读取文件工具，工具返回内容，模型在下一次请求中依据内容组织答案。工具读到文件的那一步不等于最终回答已经完成。

### _execute_tool() 方法：工具执行（含 save_memory 拦截）

原教程把工具执行及 `save_memory` 特判写在此处。新版没有照搬这套私有方法，应读 `tools/execution.py` 与 `ToolRegistry.execute()`。原问题保留，答案改为实际分层：Runner 组织调用，执行层处理调度，Registry 校验并调用工具。

源码：[nanobot/agent/tools/registry.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/registry.py#L197)，`ToolRegistry.execute`。

流程伪代码：

```text
模型返回 tool_calls
    → 执行层结合工具并发属性安排调用
    → Registry.prepare_call(name, params) 查找、转换、校验
    → tool.execute(**params)
    → 形成工具结果消息，交回 Runner
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

这个方法的主线是按条件收集 `parts`，最后连接。保留最容易理解的真实开头：

```python
root = workspace or self.workspace
parts = [self._get_identity(channel=channel, workspace=root)]

bootstrap = self._load_bootstrap_files(root)
if bootstrap:
    parts.append(bootstrap)
```

后续内容按以下顺序继续组装：

流程伪代码：

```text
身份 → bootstrap 文件 → 工具契约
    → 当前项目位置（与默认工作区不同时）
    → 长期记忆（include_memory 且不是模板内容）
    → always 技能全文、其余技能摘要
    → 归档会话摘要（存在有效摘要时）
最后用分隔符连接 parts
```

**关键点解读：**`workspace or self.workspace` 允许本轮指定项目；bootstrap 是启动说明来源，不能与消息历史混同。记忆和技能有加载条件，因此“磁盘上有文件”不等于“每次都全文注入”。always 技能加载正文，其余技能先提供摘要，体现按需获取知识。

### build_messages() 方法

新版中它是兼容包装器，先构造 `TranscriptInput` 并调用 `build_transcript()`，然后按条件合并相邻同角色消息。核心运行路径阅读时还要进入 transcript 构建，而不是只盯这个旧入口名。

源码：[nanobot/agent/context.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/context.py#L223)，`ContextBuilder.build_messages`。

流程伪代码：

```text
history、current_message、media、summary、runtime_context_blocks
    → TranscriptInput
    → build_transcript(...)
    → build_messages 的兼容处理：必要时合并相邻同角色消息
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

```python
def read_memory(self) -> str:
    return self.read_file(self.memory_file)
```

**关键点解读：**读取目标由 `self.memory_file` 决定，返回文本后是否放进系统提示词由 ContextBuilder 的加载条件决定。文件读取与提示词注入是两个步骤。

### save_memory() 方法

原教程的“保存长期记忆”主题保留；新版实际方法名是 `write_memory()`：

```python
def write_memory(self, content: str) -> None:
    self.memory_file.write_text(content, encoding="utf-8")
```

**关键点解读：**这是把给定文本写到记忆文件的存储动作，不负责自己判断什么值得记住，也不等于模型天然拥有可靠长期记忆。内容选择、整理和后续注入仍由外围流程承担。

### append_history() 方法

源码：[nanobot/agent/memory.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py#L269)，`MemoryStore.append_history`。

流程伪代码：

```text
append_history(entry: str, ...):
    在历史写入保护下处理条目及游标
    以 JSONL 格式追加历史记录
    返回对应游标
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

源码：[nanobot/agent/subagent.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/subagent.py#L609)，`SubagentManager.spawn`。

流程伪代码：

```text
接收子任务说明和来源信息
    → 校验并建立任务记录
    → 进入受并发上限约束的调度路径
    → 子任务 Runner 执行模型工具循环
    → 更新完成、失败或取消状态，按来源回传结果
```

**关键点解读：**创建任务不代表立即完成；达到并发上限后需要排队；取消、查询与结果回传都需要明确的任务标识。当前配置默认并发子任务数为 4，迭代预算来自配置，不要沿用旧示例固定 15 轮的说法。

**例子：**主任务让子任务分别阅读两个文件，创建后可以继续处理协调工作，最终根据任务状态取得结果。不能把一次创建返回值当作两个文件都已分析完毕。

<a id="section-4-7"></a>

## 4.7 工具系统 - tools/

### 4.7.1 ToolRegistry - registry.py

Registry 是“名称到工具实例”的索引，同时承担调用准备与执行入口。注册不是执行；模型看到的定义也不是 Python 函数本身。

源码：[nanobot/agent/tools/registry.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/registry.py#L29)，`ToolRegistry`。

```python
tool, params, error = self.prepare_call(name, params)
if error:
    return ToolResult.error(with_tool_error_hint(str(error)))
```

**关键点解读：**`prepare_call` 查找工具并规范、校验参数；错误先返回 `ToolResult.error`；只有通过后才进入 `execute`。源码后半段还处理工具返回的错误及异常。学习重点是“校验在副作用前”，不是背错误提示文字。

### 4.7.2 MCPToolWrapper - mcp.py

源码：[nanobot/agent/tools/mcp.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/mcp.py#L743)，`MCPToolWrapper.execute`。

流程伪代码：

```text
模型调用本地已注册的 MCP 包装工具
    → Wrapper 通过 MCPProvider 的连接和恢复机制请求远端工具
    → MCP session.call_tool(...)
    → 把返回内容和 isError 转换为本地工具结果
```

**关键点解读：**包装器让 Runner 不必分别理解每个远端服务；连接生命周期由 Provider 管理，不能每次工具调用临时拼一个裸 HTTP 请求。远端执行返回错误也需要保持错误语义，不能把它当成功文本。

### 4.7.3 MessageTool - message.py

源码：[nanobot/agent/tools/message.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/message.py#L141)，`MessageTool.execute`。

MessageTool 允许模型主动发送消息。读源码时跟踪三件事：目标来自参数还是本轮上下文，内容怎样变成出站消息，交付结果怎样反馈给工具调用方。

**关键点解读：**工具发送消息属于一次副作用，Runner 的最终回复属于轮次输出，两者不是同一个动作。实现还需记录已发送内容等状态，避免把主动发送与最终交付混为一谈。不要在客户端重写这套后端交付规则。

### 4.7.4 SpawnTool - spawn.py

这里保留原教程“用工具创建子任务”的位置。新版文件为 `tools/subagent.py`，类为 `SubagentTool`，工具名为 `subagent`。

源码：[nanobot/agent/tools/subagent.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/subagent.py#L86)，`SubagentTool.execute`。

流程伪代码：

```text
action=create → 创建子任务
action=send   → 给已有子任务发送输入
action=check  → 查询状态
action=cancel → 请求取消
```

**关键点解读：**模型通过统一工具接口提出管理任务的请求，SubagentManager 才负责真实调度。新版已不只是一个 `spawn(task)` 动作，阅读时要分清工具层的动作分派与管理器层的任务执行。

<a id="section-4-8"></a>

## 4.8 MessageBus - bus/

### 核心类：MessageBus

消息总线把平台输入与 Agent 执行解耦。新版源码位于 `bus/queue.py`，输入消息结构在 `bus/events.py`。

源码：[nanobot/bus/queue.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/bus/queue.py#L44)，`MessageBus.consume_inbound`。

```python
async def consume_inbound(self) -> InboundMessage:
    """Consume the next inbound message (blocks until available)."""
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

源码：[nanobot/channels/telegram/runtime.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/channels/telegram/runtime.py#L1943)，`TelegramChannel._on_message`。

流程伪代码：

```text
Telegram update
    → 提取文本、媒体、用户与话题信息
    → 按平台规则得到会话地址和元数据
    → 发布统一 InboundMessage
后端返回出站事件 → Telegram 适配器调用平台 SDK 发送
```

**关键点解读：**一个平台群聊可能有话题，不能仅凭用户 ID 认定会话；媒体下载与模型执行也不是同一职责。阅读这个例子时，重点找“平台消息变成统一消息”的边界。

### 飞书适配器示例

源码：[nanobot/channels/feishu/runtime.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/channels/feishu/runtime.py#L2547)，`FeishuChannel._on_message`。

飞书同样把平台事件转换为统一输入，但需要处理自己的事件结构、提及、群聊与回复地址。与 Telegram 对照阅读：平台特有解析不同，进入消息总线之后共享的执行链路相同。新增平台的主要工作应落在适配层，而不是复制整个 ReAct 循环。

<a id="section-4-10"></a>

## 4.10 配置系统 - config/schema.py

### 配置数据类

源码：[nanobot/config/schema.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/config/schema.py#L479)，`Config`。

配置使用 Pydantic 模型表达类型、默认值和嵌套结构。原教程的 YAML 讲解改为当前 `config.json`，但“配置声明与配置加载分开”的学习重点不变。

**关键点解读：**先在 Schema 查字段是否合法，再到调用方查它何时生效。界面显示的模型预设、运行时选择的 Provider、工具自己的配置有各自的装配位置，不能只读一个扁平字段表。

### 配置加载

源码：[nanobot/config/loader.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/config/loader.py#L36)，`load_config`。

流程伪代码：

```text
定位配置文件
    → 读取 JSON，处理兼容与规范化
    → Config.model_validate(...) 得到类型化配置
    → 启动层把配置传给对应运行时组件
```

**关键点解读：**JSON 能解析只证明语法有效，Pydantic 校验通过才说明满足配置结构；API Key 可用、模型服务可达仍需实际请求验证。`${VAR}` 是配置环境变量引用语法，不是把 PowerShell 的 `$env:VAR` 原样填进 JSON。

<a id="section-4-11"></a>

## 4.11 关键代码片段解读

### 片段 1：完整的消息处理链路

流程伪代码：

```text
渠道输入 → MessageBus → AgentLoop.run / _dispatch_inbound
    → 会话协调 → _process_message
    → restore → compact → command → build → run → save → respond
    → Runner 请求模型、执行工具并产生结果
    → 会话提交和出站交付
```

**关键点解读：**这是阶段关系图，不是所有请求都会走完的强制调用栈。例如命令可能提前结束。先用这条线定位某个功能归谁管，再进入具体方法；不要在多个章节重复粘贴整段单轮编排代码。

### 片段 2：MCP 工具注册流程

源码：[nanobot/agent/tools/mcp.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/mcp.py#L1425)，`MCPProvider.connect`。

流程伪代码：

```text
读取 MCP 服务配置 → Provider 连接并发现远端工具
    → 为工具建立 Wrapper 并注册
    → 模型看见工具定义
    → 调用时通过 Wrapper 进入已管理的 MCP 会话
```

**关键点解读：**发现阶段让模型知道“可以做什么”，调用阶段才真的执行。服务名、远端工具名与本地注册名需要在调试时对应起来；不要把连接成功当作每个工具调用都已成功。

### 片段 3：Provider 注册表与工厂

源码：[nanobot/providers/routing.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/providers/routing.py#L1)。

源码：[nanobot/providers/factory.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/providers/factory.py#L1)。

流程伪代码：

```text
模型与配置 → 路由决定供应商/适配器
    → 工厂构造 LLMProvider 实例
    → Runner 通过统一接口请求模型
    → 适配器把服务响应转换为统一结果
```

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
