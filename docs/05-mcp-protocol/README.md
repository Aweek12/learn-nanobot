# 05 - MCP协议与当前集成

> 已按本地 nanobot `0.4.1`、提交 `27f1a836b5c04a30763dd91960059f81c05bf505` 核对。核心源码使用中文 # 注释解读，配置示例单独标注。详见 [版本核对记录](../LOCAL_SOURCE_BASELINE.md)。

保留原协议概念与JSON报文教学示意；报文不是当前运行时源码或完整SDK实现，字段/错误码须按具体协商协议理解。项目集成、配置、包装器和练习按当前代码替换，不沿用旧版本年表。

## 目录

- [5.1 MCP 协议概述](#section-5-1)
- [5.2 核心问题：M×N 困境](#section-5-2)
- [5.3 三角架构](#section-5-3)
- [5.4 三大核心原语](#section-5-4)
- [5.5 通信协议与传输层](#section-5-5)
- [5.6 MCP vs Function Calling](#section-5-6)
- [5.7 当前连接与生命周期](#section-5-7)
- [5.8 传输与配置](#section-5-8)
- [5.9 工具名称、Schema 与重试](#section-5-9)
- [5.10 独立教学 Server 示例](#section-5-10)
- [5.11 配置与验证步骤](#section-5-11)
- [5.12 面试自查](#section-5-12)

<a id="section-5-1"></a>

## 5.1 MCP 协议概述

### 基本信息

| 项目 | 内容 |
|------|------|
| **全称** | Model Context Protocol（模型上下文协议） |
| **发布方** | Anthropic（Claude 的开发公司） |
| **发布时间** | 2024 年 11 月 |
| **协议类型** | 开放标准协议 |
| **通信协议** | JSON-RPC 2.0 |
| **传输方式** | stdio / HTTP+SSE / Streamable HTTP |
| **目标** | 标准化 AI 模型与外部工具/数据的交互方式 |

### 一句话定义

> **MCP 是 AI 应用连接外部工具和数据源的标准化协议，就像 USB-C 是设备连接的标准化接口一样。**

### 为什么 MCP 重要？

在 MCP 出现之前，每个 AI 应用（如 Claude Desktop、ChatGPT、各种 Agent 框架）都需要为每个工具和数据源编写专门的集成代码。这导致了大量的重复工作和碎片化的生态。

MCP 的出现就像当年 USB 接口统一了各种外设接口一样——它为 AI 应用提供了一个标准化的"接口协议"，让工具只需要实现一次 MCP 接口，就能被所有支持 MCP 的 AI 应用使用。

```
┌──────────────────────────────────────────────────────────┐
│                    类比：AI 界的 USB-C                     │
│                                                          │
│  USB-C 之前：                    MCP 之前：                │
│  ┌──────┐                       ┌──────────┐             │
│  │ 设备 │ → 专用线缆 → 电脑      │ AI 应用  │ → 专用代码    │
│  │ 设备 │ → 另一种线缆 → 电脑    │ AI 应用  │ → 另一套代码  │
│  │ 设备 │ → 又一种线缆 → 电脑    │ AI 应用  │ → 又一套代码  │
│  └──────┘                       └──────────┘             │
│                                                          │
│  USB-C 之后：                    MCP 之后：                │
│  ┌──────┐                       ┌──────────┐             │
│  │ 设备 │ ─┐                    │ AI 应用  │ ─┐           │
│  │ 设备 │ ─┤ USB-C ─ 电脑       │ AI 应用  │ ─┤ MCP ─ 工具│
│  │ 设备 │ ─┘                    │ AI 应用  │ ─┘           │
│  └──────┘                       └──────────┘             │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

![MCP 协议漫画](../../comics/05-mcp-protocol.png)

*MCP 就像任意门：统一标准连接所有工具，从 M×N 到 M+N*


<a id="section-5-2"></a>

## 5.2 核心问题：M×N 困境

### 问题描述

在 MCP 出现之前，AI 生态面临一个经典的 **M×N 连接问题**：

- **M 个 AI 应用**：Claude Desktop、ChatGPT、Cursor、Nanobot、LangChain 等
- **N 个工具/数据源**：GitHub、Slack、数据库、文件系统、搜索引擎等

如果每个应用都要为每个工具编写专门的集成代码，总共需要 **M × N** 个集成。

### M×N vs M+N

```
没有 MCP（M×N 连接）：
每个 AI 应用需要为每个工具单独写集成代码

AI 应用              工具/数据源
┌──────────┐        ┌──────────┐
│  Claude  │───────→│  GitHub  │
│  Desktop │──┐ ┌──→│          │
└──────────┘  │ │   └──────────┘
              │ │
┌──────────┐  │ │   ┌──────────┐
│  ChatGPT │──┼─┼──→│  Slack   │
│          │──┤ ├──→│          │
└──────────┘  │ │   └──────────┘
              │ │
┌──────────┐  │ │   ┌──────────┐
│ Nanobot  │──┘ └──→│  数据库  │
│          │───────→│          │
└──────────┘        └──────────┘

3 个应用 × 3 个工具 = 9 个集成

─────────────────────────────────────────

有 MCP（M+N 连接）：
每个应用实现 MCP Client，每个工具实现 MCP Server

AI 应用           MCP 协议        MCP Server
┌──────────┐                    ┌──────────┐
│  Claude  │──┐                 │  GitHub  │
│  Desktop │  │   ┌────────┐   │  Server  │──┐
└──────────┘  ├──→│  MCP   │←──┤          │  │
              │   │ Protocol│   └──────────┘  │
┌──────────┐  │   │        │   ┌──────────┐  │
│  ChatGPT │──┤   │标准化   │   │  Slack   │  │
│          │  │   │交互协议 │   │  Server  │──┤
└──────────┘  │   │        │   │          │  │
              │   └────────┘   └──────────┘  │
┌──────────┐  │                ┌──────────┐  │
│ Nanobot  │──┘                │  数据库  │   │
│          │                   │  Server  │──┘
└──────────┘                   └──────────┘

3 个应用 + 3 个工具 = 6 个实现（而不是 9 个）
```

**当规模增大时优势更明显**：

| 应用数(M) | 工具数(N) | 无 MCP (M×N) | 有 MCP (M+N) | 节省 |
|-----------|-----------|-------------|-------------|------|
| 3 | 3 | 9 | 6 | 33% |
| 5 | 10 | 50 | 15 | 70% |
| 10 | 20 | 200 | 30 | 85% |
| 20 | 50 | 1000 | 70 | 93% |
| 50 | 100 | 5000 | 150 | 97% |


<a id="section-5-3"></a>

## 5.3 三角架构

MCP 定义了三种角色，构成了一个清晰的三角架构：

```
┌───────────────────────────────────────────────────────────┐
│                    MCP 三角架构                             │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐  │
│  │                   MCP Host                          │  │
│  │          (AI 应用 / Agent 框架)                      │  │
│  │                                                     │  │
│  │  ┌───────────────┐                                  │  │
│  │  │               │  例如：                           │  │
│  │  │   MCP Client  │  · Claude Desktop                │  │
│  │  │   (协议客户端) │  · Cursor IDE                    │  │
│  │  │               │  · Nanobot                       │  │
│  │  └───────┬───────┘  · ChatGPT                       │  │
│  │          │                                          │  │
│  └──────────┼──────────────────────────────────────────┘  │
│             │                                             │
│             │  JSON-RPC 2.0                               │
│             │  (stdio / HTTP+SSE / Streamable HTTP)       │
│             │                                             │
│  ┌──────────▼──────────────────────────────────────────┐  │
│  │                  MCP Server                         │  │
│  │             (工具/数据提供者)                         │  │
│  │                                                     │  │
│  │  例如：                                              │  │
│  │  · @anthropic/mcp-server-filesystem (文件系统)       │  │
│  │  · @anthropic/mcp-server-github (GitHub)            │  │
│  │  · @anthropic/mcp-server-postgres (数据库)          │  │
│  │  · 任何人编写的自定义 MCP Server                     │  │
│  └─────────────────────────────────────────────────────┘  │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### 三种角色详解

| 角色 | 职责 | 类比 | Nanobot 中的对应 |
|------|------|------|-----------------|
| **MCP Host** | 运行 AI 模型的宿主应用 | 电脑主机 | Nanobot 框架本身 |
| **MCP Client** | Host 内部的协议客户端 | USB 控制器 | MCPClient 类 |
| **MCP Server** | 提供工具/数据的服务 | USB 外设 | 外部 MCP Server 进程 |

### 重要概念澄清

**MCP Client 不是独立的应用**——它是嵌入在 MCP Host 内部的协议处理模块。一个 Host 可以有多个 Client，每个 Client 连接一个 Server。

```
Nanobot (Host)
├── MCP Client 1 ──→ filesystem MCP Server
├── MCP Client 2 ──→ github MCP Server
└── MCP Client 3 ──→ database MCP Server
```


<a id="section-5-4"></a>

## 5.4 三大核心原语

MCP 定义了三种核心原语（Primitive），代表了三种不同类型的交互：

```
┌──────────────────────────────────────────────────────────┐
│                  MCP 三大核心原语                          │
│                                                          │
│  ┌─────────────────────────────────────────────────────┐ │
│  │                                                     │ │
│  │  1. Tools (工具)                                    │ │
│  │     "可执行的操作"                                   │ │
│  │     控制方: 模型决定调用                              │ │
│  │     例如: read_file, search, create_issue           │ │
│  │                                                     │ │
│  ├─────────────────────────────────────────────────────┤ │
│  │                                                     │ │
│  │  2. Resources (资源)                                │ │
│  │     "可读取的数据"                                   │ │
│  │     控制方: 应用决定读取                              │ │
│  │     例如: file://readme.md, db://users/123          │ │
│  │                                                     │ │
│  ├─────────────────────────────────────────────────────┤ │
│  │                                                     │ │
│  │  3. Prompts (提示模板)                               │ │
│  │     "可复用的提示词模板"                              │ │
│  │     控制方: 用户选择使用                              │ │
│  │     例如: code_review, summarize, translate          │ │
│  │                                                     │ │
│  └─────────────────────────────────────────────────────┘ │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 5.4.1 Tools（工具）

**定义**：MCP Server 暴露的可执行操作，由 AI 模型（LLM）自主决定是否调用。

**特征**：
- 模型控制：LLM 根据用户意图自主决定调用
- 有副作用：可能修改外部状态（写文件、发消息、创建 issue）
- 需要审慎执行：某些操作需要人类确认

**示例**：

```json
{
  "name": "read_file",
  "description": "Read the contents of a file from the filesystem",
  "inputSchema": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "The path of the file to read"
      }
    },
    "required": ["path"]
  }
}
```

**调用流程**：

```
1. MCP Client 通过 tools/list 获取可用工具列表
2. 工具描述被注入到 LLM 的 system prompt
3. LLM 根据用户意图决定调用某个工具
4. MCP Client 向 MCP Server 发送 tools/call 请求
5. MCP Server 执行工具并返回结果
6. 结果返回给 LLM 继续推理
```

### 5.4.2 Resources（资源）

**定义**：MCP Server 暴露的可读取数据，由应用程序（Host）决定何时读取。

**特征**：
- 应用控制：由 Host 应用决定读取，不是 LLM 决定
- 只读：不修改外部状态
- 类似 RESTful API 的 GET 请求

**示例**：

```json
{
  "uri": "file:///workspace/README.md",
  "name": "README.md",
  "description": "Project README file",
  "mimeType": "text/markdown"
}
```

**与 Tools 的区别**：

| 维度 | Tools | Resources |
|------|-------|-----------|
| 谁决定调用 | LLM 自主决定 | 应用程序决定 |
| 是否有副作用 | 可能有（写操作） | 没有（只读） |
| 类比 | API 的 POST/PUT | API 的 GET |
| 用途 | 执行操作 | 提供上下文数据 |

### 5.4.3 Prompts（提示模板）

**定义**：MCP Server 提供的可复用提示词模板，由用户选择使用。

**特征**：
- 用户控制：由用户在 UI 中选择使用
- 参数化：可以接受参数来定制提示
- 复用性：一次定义，多次使用

**示例**：

```json
{
  "name": "code_review",
  "description": "Review code for bugs and improvements",
  "arguments": [
    {
      "name": "code",
      "description": "The code to review",
      "required": true
    },
    {
      "name": "language",
      "description": "Programming language",
      "required": false
    }
  ]
}
```

### 三者对比表

| 维度 | Tools | Resources | Prompts |
|------|-------|-----------|---------|
| **控制方** | 模型 (LLM) | 应用 (Host) | 用户 |
| **主要用途** | 执行操作 | 提供数据 | 提示模板 |
| **有无副作用** | 可能有 | 无 | 无 |
| **调用方式** | LLM 生成 tool_call | 应用代码调用 | 用户 UI 选择 |
| **列举 API** | `tools/list` | `resources/list` | `prompts/list` |
| **执行 API** | `tools/call` | `resources/read` | `prompts/get` |
| **类比** | 动词（做事） | 名词（数据） | 模板（套路） |


<a id="section-5-5"></a>

## 5.5 通信协议与传输层

### JSON-RPC 2.0

MCP 使用 **JSON-RPC 2.0** 作为消息格式。每条消息都是一个 JSON 对象：

**请求格式**：
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "read_file",
    "arguments": {
      "path": "/workspace/README.md"
    }
  }
}
```

**响应格式**：
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "# My Project\n\nThis is the README..."
      }
    ]
  }
}
```

**错误格式**：
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32600,
    "message": "Invalid Request"
  }
}
```

### 传输层选项

MCP 支持三种传输方式，适用于不同场景：

```
┌──────────────────────────────────────────────────────────┐
│                   MCP 传输层选项                           │
│                                                          │
│  ┌───────────────────────────────────────────────────┐   │
│  │  1. stdio（标准输入/输出）                          │   │
│  │                                                   │   │
│  │  Host ←─ stdin/stdout ─→ Server 进程              │   │
│  │                                                   │   │
│  │  适用场景：本地工具                                 │   │
│  │  优点：简单、安全（进程隔离）、无需网络              │   │
│  │  缺点：只能本地，Server 与 Host 同机               │   │
│  │                                                   │   │
│  │  配置示例：                                        │   │
│  │  command: "npx @anthropic/mcp-server-filesystem"   │   │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
│  ┌───────────────────────────────────────────────────┐   │
│  │  2. HTTP + SSE（Server-Sent Events）              │   │
│  │                                                   │   │
│  │  Host ── HTTP POST → Server                       │   │
│  │  Host ←─ SSE Stream ─ Server                      │   │
│  │                                                   │   │
│  │  适用场景：远程工具                                 │   │
│  │  优点：支持远程、可穿越防火墙                       │   │
│  │  缺点：SSE 是单向的、需要两个通道                   │   │
│  │                                                   │   │
│  │  配置示例：                                        │   │
│  │  url: "https://mcp.example.com/sse"               │   │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
│  ┌───────────────────────────────────────────────────┐   │
│  │  3. Streamable HTTP（新标准，2025+）               │   │
│  │                                                   │   │
│  │  Host ←─ HTTP Stream ─→ Server                    │   │
│  │                                                   │   │
│  │  适用场景：统一的远程通信                           │   │
│  │  优点：简化了 HTTP+SSE 的双通道问题                 │   │
│  │  缺点：较新，部分实现尚未支持                       │   │
│  │                                                   │   │
│  │  配置示例：                                        │   │
│  │  url: "https://mcp.example.com/mcp"               │   │
│  └───────────────────────────────────────────────────┘   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### 传输方式对比

| 维度 | stdio | HTTP + SSE | Streamable HTTP |
|------|-------|------------|-----------------|
| **部署位置** | 本地同机 | 可远程 | 可远程 |
| **安全性** | 进程隔离，高 | 需 HTTPS | 需 HTTPS |
| **性能** | 高（无网络开销） | 中 | 中 |
| **复杂度** | 低 | 中（双通道） | 低 |
| **状态** | 有状态 | 有状态 | 可无状态 |
| **适用场景** | 开发/个人 | 团队/企业 | 未来标准 |

报文为概念示例：JSON-RPC协议错误与tools/call业务失败不同，后者可能以isError和内容返回；不要将所有文件不存在都映射为Invalid Request。


<a id="section-5-6"></a>

## 5.6 MCP vs Function Calling

这是面试中**高频考题**。很多面试官会问："MCP 和 Function Calling 有什么区别？"

### 全面对比表

| 维度 | MCP | Function Calling |
|------|-----|-----------------|
| **定义** | 标准化的工具调用协议 | LLM 原生的函数调用能力 |
| **提出方** | Anthropic（开放标准） | OpenAI（后各厂商跟进） |
| **层级** | 应用层协议（Client-Server） | 模型能力（API 参数） |
| **标准化** | 统一标准，跨框架复用 | 各 LLM 厂商实现有差异 |
| **工具发现** | 动态：`tools/list` 运行时获取 | 静态：开发时定义 |
| **工具执行** | Server 端执行 | 客户端代码执行 |
| **传输方式** | stdio / HTTP / Streamable HTTP | N/A（API 内嵌） |
| **跨应用复用** | 一个 MCP Server 服务多个应用 | 每个应用需要自己实现 |
| **生态** | 统一的 MCP Server 生态 | 碎片化 |
| **适用场景** | Agent 框架、IDE、AI 助手 | 单次 API 调用 |

### 形象对比

```
Function Calling：
┌──────────┐
│ 你的应用  │
│          │
│  ┌──────┐│  "嘿 GPT，这里有个函数 get_weather(city)，需要的话调用它"
│  │ LLM  ││──→ LLM 返回: {"name": "get_weather", "args": {"city": "北京"}}
│  └──────┘│
│          │
│  你的代码 │──→ 你自己执行 get_weather("北京")
│  执行函数 │──→ 返回结果给 LLM
└──────────┘

→ 工具定义和执行都在你的应用内部
→ 换一个应用需要重新实现

─────────────────────────────────────────

MCP：
┌──────────┐           ┌──────────────────┐
│ 你的应用  │           │  MCP Server      │
│ (Host)   │           │  (天气服务)       │
│          │  tools/   │                  │
│  MCP     │──list───→ │  返回工具列表     │
│  Client  │           │                  │
│          │  tools/   │                  │
│          │──call───→ │  执行并返回结果   │
└──────────┘           └──────────────────┘

→ 工具定义和执行都在 MCP Server 中
→ 任何支持 MCP 的应用都可以使用这个 Server
```

### 关键区别总结

**1. Function Calling 是 "模型能力"，MCP 是 "应用协议"**

Function Calling 是 LLM API 的一个功能参数——你告诉模型有哪些函数可以调用，模型决定调用哪个并生成参数。但函数的实际执行需要你自己在代码中实现。

MCP 是一个完整的 Client-Server 协议——工具的定义和执行都在 MCP Server 中，Client 只需要遵循协议就能使用。

**2. MCP 解决的是"生态复用"问题**

Function Calling：每个应用都需要自己定义和实现工具。 MCP：一个 MCP Server 编写一次，所有支持 MCP 的应用都能使用。

**3. 两者不是替代关系，而是互补**

在实际的 Agent 框架中（如 Nanobot），MCP 和 Function Calling 是配合使用的：

```
1. MCP Client 从 MCP Server 获取工具列表
2. 将工具描述转换为 Function Calling 格式
3. 通过 Function Calling 让 LLM 决定调用哪个工具
4. 框架将 LLM 的决定转换为 MCP 的 tools/call 请求
5. MCP Server 执行并返回结果
```


<a id="section-5-7"></a>

## 5.7 当前连接与生命周期

MCPProvider 由应用组装处拥有，创建/连接后与 AgentLoop 共享 ToolRegistry，关闭时执行 aclose。它支持重载与动态注册。Runner 只消费工具契约，不负责每轮建立全部MCP连接。

MCPToolWrapper、MCPResourceWrapper、MCPPromptWrapper 将不同能力适配成本地Tool。协议解析和连接失败有专门路径，不应以手写几十行JSON循环冒充完整MCP握手。

源码：[nanobot/agent/tools/mcp.py:1451](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/mcp.py#L1451)

```python
# 只选择已配置、当前缺失且不在等待授权的服务。
missing_servers = {
    name: cfg
    for name, cfg in configured_missing.items()
    if name not in authorization_pending
}
# 没有需要连接的服务时提前返回。
if not missing_servers:
    return
# 先发布 connecting 状态，实际连接在后续代码执行。
self._set_runtime_status(missing_servers, "connecting")
```

**关键点解读：**

连接属于应用资源的生命周期，工具调用属于一次执行。将它们分开，可以复用已建立的会话，并在退出时统一释放资源。调试时先确认发现了哪些工具，再检查某个工具的一次调用，避免把两类失败混在一起。

源码：[nanobot/agent/tools/mcp.py:743](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/mcp.py#L743)，`MCPToolWrapper.execute`。

```python
# MCP 工具通过受管理的会话和恢复入口发送请求。
result = await self._request_with_recovery(
    lambda session: session.call_tool(self._original_name, arguments=kwargs),
    self._tool_timeout,
    "tool",
)
```

**关键点解读：**

1. `_original_name` 是远端工具名；模型看到的本地注册名不一定与它完全相同。
2. `kwargs` 是本次工具参数；`session.call_tool` 才把请求交给 MCP 会话。
3. `_request_with_recovery` 负责受管理的请求与恢复路径，`_tool_timeout` 限制工具等待时间。
4. 后续代码把 MCP 内容转换成本地工具结果，并保留 `isError` 的失败语义。远端返回内容不代表调用成功。


<a id="section-5-8"></a>

## 5.8 传输与配置

当前 config.json **局部配置示例**：合并到已有配置，不覆盖整份文件；占位参数需替换。

```json
{
  "tools": {
    "mcpServers": {
      "learning": {
        "command": "E:/codexwork/nanobot/.venv/Scripts/python.exe",
        "args": [
          "E:/codexwork/nanobot-playground/weather_server.py"
        ],
        "toolTimeout": 30,
        "enabledTools": [
          "demo_temperature"
        ]
      }
    }
  }
}
```

command 是可执行文件，args 单独列出参数；路径需按实际机器替换。MCPServerConfig.type 可为 stdio、sse、streamableHttp。注意SDK的transport拼写可能为streamable-http，两套字段不直接互换。远程服务还需核对URL、认证、网络范围与协议协商。


<a id="section-5-9"></a>

## 5.9 工具名称、Schema 与重试

Wrapper 构造 mcp_{server}_{tool} 形式并做名称清理，参数来自inputSchema并经过规范化。不能声称始终原样拼接即全局唯一，也不能笼统说OpenAI不支持additionalProperties。

服务声明readOnlyHint/idempotentHint影响特定重放策略，不是客户端去重或权限授权，也不保证任意工具可并发。格式错误、isError、渲染失败有结构化结果。


<a id="section-5-10"></a>

## 5.10 独立教学 Server 示例

下面是独立教学示例，**不是 nanobot 源码**，使用当前已安装MCP SDK提供的FastMCP。返回固定测试值，不查询真实天气。自行保存到独立练习目录，再将配置路径指向它；本次文档修订不自动启动外部服务。

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("learning-weather")

@mcp.tool()
def demo_temperature(city: str) -> str:
    """Return labelled fixture data for a learning exercise, not live weather."""
    return f"测试数据：{city}，温度20摄氏度；不代表当前天气。"

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

stdout用于协议，不随意print日志混入；诊断日志使用合适的stderr/日志接口。stdio进程、HTTP服务与nanobot配置是不同层的生命周期。


<a id="section-5-11"></a>

## 5.11 配置与验证步骤

先在WebUI的Apps添加连接或合并tools.mcpServers，限制enabledTools，按提示保存/重启；确认发现能力，再进行一个小的只读调用。不要覆盖已有config.json。

不需要外部账户时，可读test_mcp_tool、test_mcp_probe、test_mcp_oauth的Mock契约。它们不证明真实服务连通。安全提示和OAuth流程以当前官方指南为准。

[docs/guides/configure-mcp-tools.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/guides/configure-mcp-tools.md)


<a id="section-5-12"></a>

## 5.12 面试自查

谁拥有连接生命周期？应用/MCPProvider。模型怎么调用MCP？先得到本地工具定义，再经Wrapper到服务。为什么接入MCP不自动安全？服务权限、程序边界、协议数据和内容可信度仍需分别处理。

## 下一章

[继续第 06 章](../06-install-and-hands-on/README.md)
