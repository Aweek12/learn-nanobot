# 09 - 多平台接入与共享客户端

> 已按本地 nanobot `0.4.1`、提交 `27f1a836b5c04a30763dd91960059f81c05bf505` 核对。真实源码、API 签名、伪代码与配置示例分别标注。详见 [版本核对记录](../LOCAL_SOURCE_BASELINE.md)。

将旧平台伪实现、JSONL检查方式和固定出站节流策略改成当前消息契约、渠道包与交付所有权。

## 目录

- [9.1 共同运行时](#section-9-1)
- [9.2 消息的真实结构](#section-9-2)
- [9.3 消息总线与出站](#section-9-3)
- [9.4 渠道契约与包发现](#section-9-4)
- [9.5 会话键与隔离](#section-9-5)
- [9.6 配置和实测流程](#section-9-6)
- [9.7 面试自查](#section-9-7)

<a id="section-9-1"></a>

## 9.1 共同运行时

WebUI、TUI、聊天平台通过适配器进入同一Python运行时。渠道负责平台身份、消息、媒体和交付，不在每个平台复制一套模型循环。前端有自己的交互状态，但会话、执行与安全归后端。


<a id="section-9-2"></a>

## 9.2 消息的真实结构

源码：[nanobot/bus/events.py:26](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/bus/events.py#L26)，`InboundMessage`。

流程伪代码：

```text
平台事件
    → InboundMessage(channel, sender_id, chat_id, content, ...)
    → session_key_override 或默认 channel:chat_id
    → MessageBus → 会话协调与执行
```

**关键点解读：**

`sender_id` 标识发送者，`chat_id` 标识对话地址，两者不能互换。正文属性叫 `content`。会话键决定输入进入哪个会话，可信的 override 可表达话题等额外范围；元数据则携带平台上下文，不能随意赋予外部输入内部控制权限。

源码：[nanobot/bus/events.py:55](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/bus/events.py#L55)，`OutboundMessage`。

流程伪代码：

```text
回复内容 + channel/chat_id → OutboundMessage
    metadata：平台回复、线程等路由信息
    event：内部事件语义
    delivery：交付请求及关联标识
渠道适配器 → 平台发送
```

**关键点解读：**

入站回答“谁向哪里说了什么”，出站回答“什么内容要送到哪里”。`event` 和 `delivery` 有运行时语义，不是任意显示文本。一次出站对象创建完成，只说明交付数据已准备好，不能等同于用户已读。

正文叫content，session_key是属性，线程等范围可经可信override传入。event/delivery与普通routing metadata有不同用途，不能把任意客户端字段当成内部可信控制。


<a id="section-9-3"></a>

## 9.3 消息总线与出站

源码：[nanobot/bus/queue.py:38](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/bus/queue.py#L38)，`MessageBus.publish_inbound`。

```python
async def publish_inbound(self, msg: InboundMessage) -> None:
    """Publish a message from a channel to the agent."""
    self.admission.check()
    # This queue is unbounded. Admission and enqueue must not yield apart.
    self.inbound.put_nowait(msg)
```

**关键点解读：**先做 admission 检查，再立即入队，中间没有 await 让出执行权，避免检查通过与真正接纳输入之间出现异步交接空隙。方法是 async 并不意味着每一行都会让出事件循环。

源码：[nanobot/bus/queue.py:54](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/bus/queue.py#L54)，`MessageBus.publish_event`。

```python
await self.publish_outbound(outbound_message_for_event(
    channel=channel, chat_id=chat_id, event=event, metadata=metadata,
))
```

**关键点解读：**

事件先转换成标准出站消息，再交给已有交付队列。总线运输事件，渠道决定怎样呈现或忽略它。这使进度事件不必由 Runner 直接调用 Telegram 或飞书 SDK，也保留了统一的交付管理位置。

publish用于本地订阅等待；publish_event排队渠道交付。ChannelManager与DeliveryOutbox管理发送、重试和结果；WebUIOutboundDelivery拥有其展示投影和流缓冲。不能用手写每100字符flush概括全部平台策略。


<a id="section-9-4"></a>

## 9.4 渠道契约与包发现

源码：[nanobot/channels/base.py:87](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/channels/base.py#L87)，`BaseChannel.start`。

流程伪代码：

```text
start():
    连接平台
    持续监听输入
    将输入经渠道处理入口转给消息总线
```

**关键点解读：**

这是基类要求具体渠道实现的长期任务，不是每来一条消息就重新登录平台。启动负责建立接收路径；Agent 如何推理和执行工具仍在共享后端。

源码：[nanobot/channels/base.py:104](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/channels/base.py#L104)，`BaseChannel.send`。

流程伪代码：

```text
send(outbound_message):
    按平台格式发送
    正常返回：适配器本次发送已完成
    失败：抛出对应异常，由渠道管理器统一应用重试策略
```

**关键点解读：**

发送完成不等于收件人已读。源码还区分尚未开始提交、目标被拒绝和交付结果未知等失败，避免把所有异常都当成“肯定没发出去”而盲目重试。基类声明契约，实际 SDK 操作在具体渠道包内。

还有stop及批量、delta等扩展接口。manager发现自包含渠道包，manifest/config/连接与可选webui/test可以属于包内。不要照抄旧telegram.py等平铺文件位置。


<a id="section-9-5"></a>

## 9.5 会话键与隔离

源码：[nanobot/session/keys.py:21](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/session/keys.py#L21)，`session_key_for_channel`。

```python
def session_key_for_channel(channel: str, chat_id: str, *, unified_session: bool = False) -> str:
    """Return the session key for a channel/chat pair."""
    if unified_session:
        return UNIFIED_SESSION_KEY
    return f"{channel}:{chat_id}"
```

**关键点解读：**默认键由渠道和聊天地址组成；统一会话模式返回公共键。相同 sender 并不必然属于同一会话，最终还要结合入口设置的 override 和命名空间。这也是跨平台消息不会仅凭用户名字就自动共享历史的原因。

默认按渠道/聊天生成，unified_session、override和会话命名空间会改变路径。线程规则由入口确定，不是任意metadata.thread_id统一追加固定字符串。渠道授权不等于多用户私有隔离。


<a id="section-9-6"></a>

## 9.6 配置和实测流程

先跑通普通WebUI对话，再从Channels中选择实际平台，按该渠道当前指南配置身份、密钥和访问策略。不要cat覆盖整份config。启用后验证输入、工具、出站结果，记录平台与会话身份。

主会话位于SQLite，不再通过ls sessions寻找telegram:xxx.jsonl来判断隔离。平台账号未配置时，用渠道包tests和根tests/channels理解Mock契约，不声称真实连通。

[docs/chat-apps.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/chat-apps.md) · [docs/channel-package-guide.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/channel-package-guide.md)


<a id="section-9-7"></a>

## 9.7 面试自查

Provider失败与Channel失败在哪里区分？前者是模型协议边界，后者是平台接入。为什么页面没显示不能直接重跑工具？执行、提交与交付可能状态不同。为什么双队列不保证简单增加消费者就可水平扩展？会话状态与资源所有权仍需协调。

## 下一章

[继续第 10 章](../10-subagent-and-cron/README.md)
