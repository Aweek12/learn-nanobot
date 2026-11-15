# 10 - 子 Agent、Cron 与心跳

> 已按本地 nanobot `0.4.1`、提交 `27f1a836b5c04a30763dd91960059f81c05bf505` 核对。核心源码使用中文 # 注释解读，配置示例单独标注。详见 [版本核对记录](../LOCAL_SOURCE_BASELINE.md)。

当前的子任务控制、调度与心跳实现有明确接口；原手写后台类与APScheduler代码不再冒充项目源码。

## 目录

- [10.1 三个不同问题](#section-10-1)
- [10.2 当前子任务API](#section-10-2)
- [10.3 子任务预算与生命周期](#section-10-3)
- [10.4 Cron并非APScheduler](#section-10-4)
- [10.5 Cron工具参数](#section-10-5)
- [10.6 Heartbeat实际注册](#section-10-6)
- [10.7 持续目标与恢复](#section-10-7)
- [10.8 自查](#section-10-8)

<a id="section-10-1"></a>

## 10.1 三个不同问题

子任务管理执行分工，Cron决定何时触发，Heartbeat执行配置的周期检查；它们不等于“主Agent是同步的所以只能做一件事”。主执行本身是异步调度，多会话、输入注入和工具并发另有机制。


<a id="section-10-2"></a>

## 10.2 当前子任务API

源码：[nanobot/agent/subagent.py:625](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/subagent.py#L625)

```python
# 创建失败转为工具错误；创建成功不表示任务已完成。
try:
    # 将任务说明、来源、运行条件交给实际创建入口。
    record = await self._create_task(
        task, label, origin_channel, origin_chat_id, session_key,
        origin_message_id, temperature, workspace_scope, runtime, announce=True, origin_turn_id=origin_turn_id,
        session_policy=session_policy,
    )
except SubagentControlError as exc:
    return ToolResult.error(f"Error: {exc}")
# 读取刚创建任务的状态，用任务 ID 关联后续查询。
status = record.status
return f"Subagent [{status.label}] started (id: {status.task_id}). I'll notify you when it completes."
```

**关键点解读：**

`task` 描述工作，来源渠道、会话与轮次信息回答“结果交给谁”，运行条件决定模型及工作区范围。任务有自己的执行过程，创建回执不能直接当作完成结果。

源码：[nanobot/agent/tools/subagent.py:91](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/subagent.py#L91)

```python
# 先处理创建动作，其余动作使用已有任务标识。
if action == "create":
    # 拒绝空任务，防止创建没有实际目标的子任务。
    if not task or not task.strip():
        return ToolResult.error("Error: create requires a non-empty task")
    # 创建和等待策略委托给专门入口。
    return await self._create_task(task, label, temperature, wait)
# 后续查询、发送和取消使用请求会话标识核对归属。
owner = current_request_session_key()
```

**关键点解读：**

动作决定哪些参数有意义：创建需要任务说明，查询和取消需要任务标识，发送需要后续输入。`wait` 影响调用等待方式，但具体能否后台运行还受宿主能力等条件影响。先理解动作与状态转换，再查看完整签名中的可选参数。

工具名subagent，action=create/send/cancel/check。wait、临时会话和宿主接收后台能力决定等待还是后台。send回执可表示已排队/已注入，不保证任务已采取行动。任务读取、取消与结果接纳检查归属。


<a id="section-10-3"></a>

## 10.3 子任务预算与生命周期

SubagentManager持有自己的Runner，运行任务有独立上下文和状态。max_iterations默认读取AgentDefaults.max_tool_iterations（当前200），max_concurrent_subagents默认4，并有活动/收件箱等容量边界。不能用固定15次解释所有递归和资源防护。

create → 接纳与状态记录 → 调度/执行 → 结果/停止状态 → 向原会话交付；中断恢复、关闭与按会话取消另有路径。部分结果或迭代停止不等于任务完成。

[nanobot/agent/subagent.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/subagent.py)


<a id="section-10-4"></a>

## 10.4 Cron并非APScheduler

源码：[nanobot/cron/service.py:768](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/cron/service.py#L768)

```python
# 规范化轮次任务并检查 Agent 绑定。
_normalize_agent_turn_job(job)
self._enforce_agent_binding(job)
# 能直接持久化时写入任务存储并更新计时器。
if self._should_persist_store():
    store = self._require_store()
    store.jobs.append(job)
    # 登记任务后保存；这里只是调度登记，不是立即跑模型。
    self._save_store()
    self._arm_timer()
# 另一条路径记录新增动作，由对应存储流程处理。
else:
    self._append_action("add", asdict(job))
```

**关键点解读：**

`schedule` 回答何时触发，`message` 回答触发后让 Agent 做什么，交付字段回答结果送往哪里。这三类信息不能互相代替。创建任务只是登记日程，不是马上执行模型调用。

当前CronService自行管理CronJob、CronSchedule、CronPayload与运行记录，不使用原文展示的AsyncIOScheduler实现。Schedule.kind为at/every/cron，at_ms/every_ms/expr/tz是存储模型；工具面向模型的参数又是另一层。


<a id="section-10-5"></a>

## 10.5 Cron工具参数

源码：[nanobot/agent/tools/cron.py:144](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/cron.py#L144)

```python
# 新增任务需要检查本轮是否已经来自 Cron。
if action == "add":
    request = current_request_context()
    # 禁止在定时执行中继续创建任务，避免递归扩张。
    if request is not None and is_cron_turn(request.metadata):
        return ToolResult.error("Error: cannot schedule new jobs from within a cron job execution")
    # 时间参数转换和登记委托给 _add_job。
    return self._add_job(name, message or "", every_seconds, cron_expr, tz, at)
# 其余动作分派到各自的管理方法。
elif action == "list":
    return self._list_jobs(include_disabled=include_disabled)
elif action == "remove":
    return self._remove_job(job_id)
elif action == "update":
    return self._update_job(job_id, name, message, every_seconds, cron_expr, tz, at)
# 启用与禁用共享入口，状态取决于 action。
elif action in {"enable", "disable"}:
    return self._set_job_enabled(job_id, enabled=action == "enable")
elif action == "history":
    return self._job_history(job_id, limit)
return f"Unknown action: {action}"
```

**关键点解读：**

模型-facing 参数便于表达“每隔多少秒”或“某个时区的 Cron 表达式”，服务内部则使用 `CronSchedule`。所以工具调用 JSON 与配置文件 JSON 不是同一种结构。下面的例子展示参数关系，重点比较 `cron_expr` 与 `tz` 如何一起定义触发时间。

下面只是**工具参数示例**，不是config.json，也不会由本次修订实际创建任务：

```json
{"action":"add","name":"学习提醒","message":"复习Runner与Loop区别","cron_expr":"0 9 * * *","tz":"Asia/Shanghai"}
```

删除/运行等操作使用实际支持的action和job_id；不要沿用嵌套schedule或按名称删除的旧示意。自动化定义、触发、会话执行与通知是不同步骤。


<a id="section-10-6"></a>

## 10.6 Heartbeat实际注册

当前 config.json **局部配置示例**：合并到已有配置，不覆盖整份文件；占位参数需替换。

```json
{
  "gateway": {
    "heartbeat": {
      "enabled": true,
      "intervalS": 1800
    }
  }
}
```

配置位于gateway.heartbeat，不在agents.defaults.heartbeat，也没有教程虚构的keep_recent_messages字段。gateway_runtime把heartbeat注册为Cron系统任务，读取HEARTBEAT.md并判断活动内容。不是每到时间无条件重放完整聊天。

[nanobot/cli/gateway_runtime.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/cli/gateway_runtime.py)


<a id="section-10-7"></a>

## 10.7 持续目标与恢复

Goal通过goal/与继续策略推进，普通轮次、持续目标、定时触发来源不同。后台gateway不等于OS开机服务；服务未运行、漏过触发或重启时不能承诺所有工作自动补跑。未知副作用不应盲目重试。

练习优先读test_subagent_lifecycle、test_subagent_control及Cron相关测试；真实提醒只在读者明确创建时操作。本章不自动注册任何任务。


<a id="section-10-8"></a>

## 10.8 自查

解释“一个子管理器一个Runner”为何仍可有多个子任务；区分send已接纳与已执行；区分CronSchedule存储字段与CronTool输入；指出心跳配置位置和系统注册者。

## 下一章

[继续第 11 章](../11-security-and-deploy/README.md)
