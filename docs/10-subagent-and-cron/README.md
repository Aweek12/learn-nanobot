# 10 - 子 Agent、Cron 与心跳

> 已按本地 nanobot `0.4.1`、提交 `27f1a836b5c04a30763dd91960059f81c05bf505` 核对。方法使用本地新版完整源码并配中文 # 注释解读，配置示例单独标注。详见 [版本核对记录](../LOCAL_SOURCE_BASELINE.md)。

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

<a id="method-subagentmanager-spawn"></a>

源码：[nanobot/agent/subagent.py:609](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/subagent.py#L609)，`SubagentManager.spawn` 完整方法。

```python
# 创建有来源信息的子任务，返回任务状态和标识；创建回执不等于执行完成。
async def spawn(
    self,
    task: str,
    label: str | None = None,
    origin_channel: str = "cli",
    origin_chat_id: str = "direct",
    session_key: str | None = None,
    origin_message_id: str | None = None,
    temperature: float | None = None,
    workspace_scope: WorkspaceScope | None = None,
    *,
    runtime: LLMRuntime | None = None,
    origin_turn_id: str | None = None,
    session_policy: SessionPolicy | None = None,
) -> str:
    """Start background work and route its terminal result to the parent."""
    try:
        # 创建子任务记录并安排执行，保留来源与运行条件。
        record = await self._create_task(
            task, label, origin_channel, origin_chat_id, session_key,
            origin_message_id, temperature, workspace_scope, runtime, announce=True, origin_turn_id=origin_turn_id,
            session_policy=session_policy,
        )
    # 处理 SubagentControlError，按本方法约定记录、转换或继续传播错误。
    except SubagentControlError as exc:
        return ToolResult.error(f"Error: {exc}")
    status = record.status
    return f"Subagent [{status.label}] started (id: {status.task_id}). I'll notify you when it completes."
```

**关键点解读：**

`task` 描述工作，来源渠道、会话与轮次信息回答“结果交给谁”，运行条件决定模型及工作区范围。任务有自己的执行过程，创建回执不能直接当作完成结果。

<a id="method-subagenttool-execute"></a>

源码：[nanobot/agent/tools/subagent.py:86](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/subagent.py#L86)，`SubagentTool.execute` 完整方法。

```python
# 按动作创建、查询、发送输入或取消子任务，并核对任务归属。
async def execute(
    self, action: str, task_id: str | None = None, message: str | None = None,
    task: str | None = None, label: str | None = None,
    temperature: float | None = None, wait: bool = False, **kwargs: Any,
) -> str:
    # 创建动作需要非空任务说明；其他动作管理已有任务。
    if action == "create":
        if not task or not task.strip():
            return ToolResult.error("Error: create requires a non-empty task")
        return await self._create_task(task, label, temperature, wait)
    # 取得调用会话身份，用于查询、发送和取消时核对任务归属。
    owner = current_request_session_key()
    try:
        if action == "check":
            if not owner:
                raise SubagentControlError("task unavailable")
            if task_id is not None:
                return json.dumps(self._manager.check(task_id, owner).as_dict(), ensure_ascii=False)
            return json.dumps({
                "tasks": [status.as_dict()
                          for status in self._manager.statuses_for_session(owner, include_history=False).values()],
            }, ensure_ascii=False)
        if action in {"send", "cancel"}:
            if not task_id:
                return ToolResult.error(f"Error: {action} requires task_id")
            if action == "send":
                return json.dumps(await self._manager.send(task_id, owner, message))
            status = await self._manager.cancel(task_id, owner)
            return json.dumps(status.as_dict(), ensure_ascii=False)
        return ToolResult.error("Error: unknown subagent action")
    # 处理 SubagentControlError，按本方法约定记录、转换或继续传播错误。
    except SubagentControlError as exc:
        return ToolResult.error(f"Error: {exc}")
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

<a id="method-cronservice-add-job"></a>

源码：[nanobot/cron/service.py:727](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/cron/service.py#L727)，`CronService.add_job` 完整方法。

```python
# 创建带日程和来源的任务记录，保存并安排后续计时。
def add_job(
    self,
    name: str,
    schedule: CronSchedule,
    message: str,
    deliver: bool = False,
    channel: str | None = None,
    to: str | None = None,
    delete_after_run: bool = False,
    channel_meta: dict[str, Any] | None = None,
    session_key: str | None = None,
    origin_channel: str | None = None,
    origin_chat_id: str | None = None,
    origin_metadata: dict[str, Any] | None = None,
) -> CronJob:
    """Add a new job."""
    # 登记日程前先检查调度参数是否合法。
    _validate_schedule_for_add(schedule)
    now = _now_ms()

    # 创建任务记录，包含日程、执行消息、来源和下次触发状态。
    job = CronJob(
        id=str(uuid.uuid4())[:8],
        name=name,
        enabled=True,
        schedule=schedule,
        payload=CronPayload(
            kind="agent_turn",
            message=message,
            deliver=deliver,
            channel=channel,
            to=to,
            channel_meta=channel_meta or {},
            session_key=session_key,
            origin_channel=origin_channel,
            origin_chat_id=origin_chat_id,
            origin_metadata=origin_metadata or {},
        ),
        state=CronJobState(next_run_at_ms=_compute_next_run(schedule, now)),
        created_at_ms=now,
        updated_at_ms=now,
        delete_after_run=delete_after_run,
    )
    _normalize_agent_turn_job(job)
    self._enforce_agent_binding(job)
    if self._should_persist_store():
        store = self._require_store()
        store.jobs.append(job)
        # 保存任务登记，不代表定时执行已经完成。
        self._save_store()
        # 根据最新日程重新安排触发计时。
        self._arm_timer()
    else:
        self._append_action("add", asdict(job))

    logger.info("Cron: added job '{}' ({})", name, job.id)
    return job
```

**关键点解读：**

`schedule` 回答何时触发，`message` 回答触发后让 Agent 做什么，交付字段回答结果送往哪里。这三类信息不能互相代替。创建任务只是登记日程，不是马上执行模型调用。

当前CronService自行管理CronJob、CronSchedule、CronPayload与运行记录，不使用原文展示的AsyncIOScheduler实现。Schedule.kind为at/every/cron，at_ms/every_ms/expr/tz是存储模型；工具面向模型的参数又是另一层。


<a id="section-10-5"></a>

## 10.5 Cron工具参数

<a id="method-crontool-execute"></a>

源码：[nanobot/agent/tools/cron.py:131](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/cron.py#L131)，`CronTool.execute` 完整方法。

```python
# 按新增、列表、删除、更新、启停和历史动作分派到对应管理入口。
async def execute(
    self,
    action: str,
    name: str | None = None,
    message: str | None = None,
    every_seconds: int | None = None,
    cron_expr: str | None = None,
    tz: str | None = None,
    at: str | None = None,
    job_id: str | None = None,
    include_disabled: bool = False,
    limit: int = 5,
) -> str:
    # 新增 Cron 任务先核对当前输入来源，防止定时任务递归创建任务。
    if action == "add":
        request = current_request_context()
        if request is not None and is_cron_turn(request.metadata):
            return ToolResult.error("Error: cannot schedule new jobs from within a cron job execution")
        return self._add_job(name, message or "", every_seconds, cron_expr, tz, at)
    elif action == "list":
        return self._list_jobs(include_disabled=include_disabled)
    elif action == "remove":
        return self._remove_job(job_id)
    elif action == "update":
        return self._update_job(job_id, name, message, every_seconds, cron_expr, tz, at)
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
