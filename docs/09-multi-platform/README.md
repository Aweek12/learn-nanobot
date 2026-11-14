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

**真实源码节选**：[nanobot/bus/events.py:26](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/bus/events.py#L26)，`InboundMessage`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
@dataclass
class InboundMessage:
    """Message received from a chat channel."""

    channel: str
    sender_id: str
    chat_id: str
    content: str
    timestamp: datetime = field(default_factory=datetime.now)
    media: list[str] = field(default_factory=list)  # Media paths or URLs
    metadata: dict[str, Any] = field(default_factory=dict)
    session_key_override: str | None = None  # Optional override for thread-scoped sessions
    require_existing_session: bool = False
    input_role: Literal["user", "system"] | None = None
    session_generation: str | None = None  # Trusted identity assigned at session admission

    @property
    def session_key(self) -> str:
        """Unique key for session identification."""
        return self.session_key_override or f"{self.channel}:{self.chat_id}"

    @property
    def is_user_input(self) -> bool:
        """Whether this message should enter the conversation as user input."""
        if self.input_role is not None:
            return self.input_role == "user"
        return self.channel != "system"
```

**真实源码节选**：[nanobot/bus/events.py:55](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/bus/events.py#L55)，`OutboundMessage`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
@dataclass
class OutboundMessage:
    """Message to send to a chat channel.

    ``event`` carries internal runtime/UI semantics. ``metadata`` is reserved
    for channel routing context (``message_id``, thread ids, etc.) and optional
    ``OUTBOUND_META_AGENT_UI`` blobs for rich clients.
    """

    channel: str
    chat_id: str
    content: str
    reply_to: str | None = None
    media: list[str] = field(default_factory=list)
    metadata: dict[str, Any] = field(default_factory=dict)
    buttons: list[list[str]] = field(default_factory=list)
    event: "AgentEvent | None" = None
    delivery: DeliveryRequest | None = None
    delivery_id: str | None = None
```

正文叫content，session_key是属性，线程等范围可经可信override传入。event/delivery与普通routing metadata有不同用途，不能把任意客户端字段当成内部可信控制。


<a id="section-9-3"></a>

## 9.3 消息总线与出站

**真实源码节选**：[nanobot/bus/queue.py:38](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/bus/queue.py#L38)，`MessageBus.publish_inbound`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
async def publish_inbound(self, msg: InboundMessage) -> None:
    """Publish a message from a channel to the agent."""
    self.admission.check()
    # This queue is unbounded. Admission and enqueue must not yield apart.
    self.inbound.put_nowait(msg)
```

**真实源码节选**：[nanobot/bus/queue.py:54](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/bus/queue.py#L54)，`MessageBus.publish_event`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
async def publish_event(
    self,
    event: AgentEvent,
    *,
    channel: str,
    chat_id: str,
    metadata: Mapping[str, Any] | None = None,
) -> None:
    """Queue a typed event using the existing outbound delivery contract.

    The bus transports event values without inspecting their fields. Known
    events retain their text fallback; channel adapters decide how to render
    or ignore events they receive.
    """
    await self.publish_outbound(outbound_message_for_event(
        channel=channel, chat_id=chat_id, event=event, metadata=metadata,
    ))
```

publish用于本地订阅等待；publish_event排队渠道交付。ChannelManager与DeliveryOutbox管理发送、重试和结果；WebUIOutboundDelivery拥有其展示投影和流缓冲。不能用手写每100字符flush概括全部平台策略。


<a id="section-9-4"></a>

## 9.4 渠道契约与包发现

**真实源码节选**：[nanobot/channels/base.py:87](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/channels/base.py#L87)，`BaseChannel.start`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
@abstractmethod
async def start(self) -> None:
    """
    Start the channel and begin listening for messages.

    This should be a long-running async task that:
    1. Connects to the chat platform
    2. Listens for incoming messages
    3. Forwards messages to the bus via _handle_message()
    """
    pass
```

**真实源码节选**：[nanobot/channels/base.py:104](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/channels/base.py#L104)，`BaseChannel.send`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
@abstractmethod
async def send(self, msg: OutboundMessage) -> None:
    """
    Send a message through this channel.

    Args:
        msg: The message to send.

    Implementations should raise on delivery failure so the channel manager
    can apply any retry policy in one place.
    Raise ChannelNotReadyError only before submitting any part of a message.
    Raise DeliveryRejectedError for a rejected destination or policy.
    Other exceptions have an unknown delivery outcome for scheduled results.
    A normal return confirms this adapter completed, not a recipient read receipt.
    """
    pass
```

还有stop及批量、delta等扩展接口。manager发现自包含渠道包，manifest/config/连接与可选webui/test可以属于包内。不要照抄旧telegram.py等平铺文件位置。


<a id="section-9-5"></a>

## 9.5 会话键与隔离

**真实源码节选**：[nanobot/session/keys.py:21](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/session/keys.py#L21)，`session_key_for_channel`。仅移除公共缩进，未补齐文件其余依赖，不作为独立脚本执行。

```python
def session_key_for_channel(channel: str, chat_id: str, *, unified_session: bool = False) -> str:
    """Return the session key for a channel/chat pair."""
    if unified_session:
        return UNIFIED_SESSION_KEY
    return f"{channel}:{chat_id}"
```

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
