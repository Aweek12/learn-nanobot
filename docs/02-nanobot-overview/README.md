# 02 - Nanobot 项目概览

> 已按本地 nanobot `0.4.1`、提交 `27f1a836b5c04a30763dd91960059f81c05bf505` 核对。真实源码、API 签名、伪代码与配置示例分别标注。详见 [版本核对记录](../LOCAL_SOURCE_BASELINE.md)。

本章按当前本地代码重整项目概念和能力，移除已失效的 YAML 配置、静态统计与纯 Markdown 会话模型。

## 目录

- [2.1 当前项目定位](#section-2-1)
- [2.2 运行入口与模型](#section-2-2)
- [2.3 配置示例](#section-2-3)
- [2.4 核心能力](#section-2-4)
- [2.5 学习主线与面试](#section-2-5)
- [2.6 本章依据](#section-2-6)

<a id="section-2-1"></a>

## 2.1 当前项目定位

nanobot 是自托管 Agent 工程，Python gateway 提供共享运行时，浏览器、终端、聊天渠道复用执行、会话、工具与安全。源码学习价值来自能够完整追踪机制，不依赖“仅4000行覆盖所有工业能力”等过时口号。

本地基准版本0.4.1；当前代码已包含客户端、持久化、协议、发布与平台兼容等大量模块。Stars、行数、平台数不作为能力保证，本章不保留未经重新测量的旧统计。


<a id="section-2-2"></a>

## 2.2 运行入口与模型

源码安装使用 Python3.11+、Git 和匹配客户端所需的 Bun。WebUI 用 nanobot webui，默认终端交互用 nanobot，旧Python交互可显式选 --classic。客户端不是独立Agent实现。

Provider 经元数据、路由和工厂选择兼容或专用实现。推荐在 Settings → Models 建预设并在输入区选中；Ollama 等本地服务也可接入，但工具能力和硬件需另行验证。


<a id="section-2-3"></a>

## 2.3 配置示例

当前 config.json **局部配置示例**：合并到已有配置，不覆盖整份文件；占位参数需替换。

```json
{
  "providers": {
    "ollama": {
      "apiBase": "http://localhost:11434/v1"
    }
  },
  "modelPresets": {
    "local": {
      "provider": "ollama",
      "model": "llama3.2",
      "maxTokens": 2048,
      "contextWindowTokens": 32768
    }
  },
  "agents": {
    "defaults": {
      "modelPreset": "local"
    }
  }
}
```

模型标签仅取自上游配置示例，须已在本机服务下载并可用。上下文额度应与实际服务匹配；本地推理不自动关闭网页、MCP或Shell网络能力。


<a id="section-2-4"></a>

## 2.4 核心能力

| 能力 | 当前机制 |
| --- | --- |
| 工具闭环 | AgentRunner + ToolRegistry + execution |
| 会话 | SessionState + SQLite |
| 记忆 | MEMORY.md / history.jsonl / Dream |
| 技能 | 工作区、插件、内置技能与按需加载 |
| 外部能力 | MCPProvider 与包装器 |
| 多入口 | 自包含渠道包、WebUI、TUI |
| 后台工作 | 子任务、Cron、Heartbeat、持续目标 |

这些是已有能力，不能把启用框架功能写成自己的实现。各功能是否启用和成功要分别验证。


<a id="section-2-5"></a>

## 2.5 学习主线与面试

先核对解释器、源码提交、预设和工作区，跑一句问候与一个文件任务。再看工具/Provider契约、Runner、Context、Loop、SessionState、安全与交付。

对比其他框架应指定版本、场景和证据，不用“对方50万行/固定慢多少”作结论。面试描述个人实际阅读与调试，而非预先声称工业级性能。


<a id="section-2-6"></a>

## 2.6 本章依据

[README.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/README.md) · [pyproject.toml](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/pyproject.toml) · [docs/concepts.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/concepts.md) · [docs/provider-cookbook.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/provider-cookbook.md)

## 下一章

[继续第 03 章](../03-architecture-deep-dive/README.md)
