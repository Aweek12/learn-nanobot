# 本地源码核对基准与修订记录

本次修订只更新docs，运行时代码及projects原示例不改。以当前本地检出为事实来源，不声称其他旧教程片段对应哪个历史提交。

| 项目 | 值 |
| --- | --- |
| 上游 | HKUDS/nanobot |
| Git提交 | `27f1a836b5c04a30763dd91960059f81c05bf505` |
| 项目配置版本 | `0.4.1` |
| 核对日期 | 2026-11-14 |

## 主要变化

- Loop编排与Runner对象复用，替换每消息新建的解释。
- _save_turn结构化新增历史、SQLite与SessionState，日志预览与正文分开。
- Consolidator/MemoryArchiver、ContextGovernor与Dream替代旧统一合并类叙述。
- config.json、模型预设、Python3.11+、WebUI/TUI/gateway与当前配置默认值。
- Tool/ToolRegistry真实注册、参数校验、执行和Hook接口。
- subagent控制API、当前CronService与gateway heartbeat注册。
- MCPProvider生命周期、Wrapper.execute与当前嵌套配置。
- 渠道包、消息结构、会话键与出站交付所有权。
- 路径/网络/OS沙箱分层，去除startswith/cwd完备隔离示意。
- 134题中涉及实现的回答及简历/STAR同步，保留通用题的学习作用。

## 代码标记

真实源码节选仅移除类内公共缩进；API签名省略方法体；text伪代码只表达控制流；配置块是局部合并示例；独立教学脚本不声称已部署或被纳入运行时。图与概念例子不是全部调用栈。

## 核对来源

- [Dockerfile](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/Dockerfile)
- [README.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/README.md)
- [docker-compose.bwrap.yml](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docker-compose.bwrap.yml)
- [docker-compose.yml](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docker-compose.yml)
- [docs/architecture.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/architecture.md)
- [docs/channel-package-guide.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/channel-package-guide.md)
- [docs/chat-apps.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/chat-apps.md)
- [docs/concepts.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/concepts.md)
- [docs/guides/configure-mcp-tools.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/guides/configure-mcp-tools.md)
- [docs/memory.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/memory.md)
- [docs/provider-cookbook.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/provider-cookbook.md)
- [docs/python-sdk.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/python-sdk.md)
- [docs/quick-start.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/quick-start.md)
- [docs/webui.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/webui.md)
- [nanobot/agent/context.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/context.py)
- [nanobot/agent/context_governance.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/context_governance.py)
- [nanobot/agent/hook.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/hook.py)
- [nanobot/agent/loop.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/loop.py)
- [nanobot/agent/memory.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/memory.py)
- [nanobot/agent/runner.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/runner.py)
- [nanobot/agent/skill_metadata.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/skill_metadata.py)
- [nanobot/agent/skills.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/skills.py)
- [nanobot/agent/subagent.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/subagent.py)
- [nanobot/agent/tools/__init__.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/__init__.py)
- [nanobot/agent/tools/_search_content.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/_search_content.py)
- [nanobot/agent/tools/_windows_job.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/_windows_job.py)
- [nanobot/agent/tools/apply_patch.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/apply_patch.py)
- [nanobot/agent/tools/base.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/base.py)
- [nanobot/agent/tools/cli_apps.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/cli_apps.py)
- [nanobot/agent/tools/cli_apps_config.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/cli_apps_config.py)
- [nanobot/agent/tools/context.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/context.py)
- [nanobot/agent/tools/cron.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/cron.py)
- [nanobot/agent/tools/exec_session.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/exec_session.py)
- [nanobot/agent/tools/execution.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/execution.py)
- [nanobot/agent/tools/file_state.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/file_state.py)
- [nanobot/agent/tools/filesystem.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/filesystem.py)
- [nanobot/agent/tools/filesystem_config.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/filesystem_config.py)
- [nanobot/agent/tools/image_generation.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/image_generation.py)
- [nanobot/agent/tools/image_generation_config.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/image_generation_config.py)
- [nanobot/agent/tools/loader.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/loader.py)
- [nanobot/agent/tools/mcp.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/mcp.py)
- [nanobot/agent/tools/mcp_oauth.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/mcp_oauth.py)
- [nanobot/agent/tools/message.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/message.py)
- [nanobot/agent/tools/path_utils.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/path_utils.py)
- [nanobot/agent/tools/registry.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/registry.py)
- [nanobot/agent/tools/rg.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/rg.py)
- [nanobot/agent/tools/runtime_control.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/runtime_control.py)
- [nanobot/agent/tools/sandbox.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/sandbox.py)
- [nanobot/agent/tools/sandbox_check.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/sandbox_check.py)
- [nanobot/agent/tools/sandbox_launcher.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/sandbox_launcher.py)
- [nanobot/agent/tools/schema.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/schema.py)
- [nanobot/agent/tools/search.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/search.py)
- [nanobot/agent/tools/self.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/self.py)
- [nanobot/agent/tools/self_config.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/self_config.py)
- [nanobot/agent/tools/session_messages.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/session_messages.py)
- [nanobot/agent/tools/sessions.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/sessions.py)
- [nanobot/agent/tools/shell.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/shell.py)
- [nanobot/agent/tools/shell_config.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/shell_config.py)
- [nanobot/agent/tools/shell_output.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/shell_output.py)
- [nanobot/agent/tools/subagent.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/subagent.py)
- [nanobot/agent/tools/web.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/web.py)
- [nanobot/agent/tools/web_config.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/web_config.py)
- [nanobot/bus/events.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/bus/events.py)
- [nanobot/bus/queue.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/bus/queue.py)
- [nanobot/channels/base.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/channels/base.py)
- [nanobot/channels/manager.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/channels/manager.py)
- [nanobot/cli/gateway_runtime.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/cli/gateway_runtime.py)
- [nanobot/config/loader.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/config/loader.py)
- [nanobot/config/schema.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/config/schema.py)
- [nanobot/cron/service.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/cron/service.py)
- [nanobot/gateway/runtime.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/gateway/runtime.py)
- [nanobot/providers/base.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/providers/base.py)
- [nanobot/providers/factory.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/providers/factory.py)
- [nanobot/providers/routing.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/providers/routing.py)
- [nanobot/security/network.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/security/network.py)
- [nanobot/security/workspace_access.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/security/workspace_access.py)
- [nanobot/security/workspace_policy.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/security/workspace_policy.py)
- [nanobot/session/keys.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/session/keys.py)
- [nanobot/session/recovery.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/session/recovery.py)
- [nanobot/session/state.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/session/state.py)
- [nanobot/webui/outbound_delivery.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/webui/outbound_delivery.py)
- [pyproject.toml](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/pyproject.toml)
- [tests/agent/test_runner_core.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/tests/agent/test_runner_core.py)
- [tests/agent/test_task_cancel.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/tests/agent/test_task_cancel.py)

## 许可

实质性源码节选适用上游MIT许可，Copyright (c) 2025-present Xubin Ren and the nanobot contributors；完整声明见 [UPSTREAM_LICENSE](UPSTREAM_LICENSE)。原教程许可不因此替代上游代码许可。

## 验证边界

完整机器记录见 [source-verification.json](source-verification.json)，来源哈希与节选范围见 [source-audit.json](source-audit.json)。核对84个来源文件与固定提交的一致性、23段真实源码、37个Python代码块语法、13个JSON块、6份Pydantic配置、153个页内锚点和134道题编号，均通过检查。

这不代表业务示例已运行或上游全部分支已验证。未使用个人API Key、未调用真实平台、未启动定时任务或部署业务；配置结构校验不证明认证与服务可达。原漫画和projects保持原样，不作为当前实现证据。第三方市场/薪资/框架统计未在本次源码修订中验证为当前事实。
