# 10 - 子 Agent、Cron 与心跳

> 已按本地 nanobot `0.4.1`、提交 `27f1a836b5c04a30763dd91960059f81c05bf505` 核对。真实源码、API 签名、伪代码与配置示例分别标注。详见 [版本核对记录](../LOCAL_SOURCE_BASELINE.md)。

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

源码：[nanobot/agent/subagent.py:609](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/subagent.py#L609)。

流程伪代码：

```text
spawn(task, 来源信息, 运行条件):
    建立有归属的子任务
    交给并发受限的调度路径
    执行后更新任务状态，并向来源回传结果
```

**关键点解读：**

`task` 描述工作，来源渠道、会话与轮次信息回答“结果交给谁”，运行条件决定模型及工作区范围。任务有自己的执行过程，创建回执不能直接当作完成结果。

源码：[nanobot/agent/tools/subagent.py:86](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/subagent.py#L86)。

流程伪代码：

```text
execute(action, task_id, task/message, wait, ...):
    create → 创建任务
    send   → 发送后续输入
    check  → 查询任务状态
    cancel → 请求取消
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

源码：[nanobot/cron/service.py:727](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/cron/service.py#L727)。

流程伪代码：

```text
add_job(name, schedule, message, 交付及来源信息):
    创建 CronJob 和对应 payload
    计算后续触发时间并更新任务存储
    返回任务记录
到期后由调度服务触发执行
```

**关键点解读：**

`schedule` 回答何时触发，`message` 回答触发后让 Agent 做什么，交付字段回答结果送往哪里。这三类信息不能互相代替。创建任务只是登记日程，不是马上执行模型调用。

当前CronService自行管理CronJob、CronSchedule、CronPayload与运行记录，不使用原文展示的AsyncIOScheduler实现。Schedule.kind为at/every/cron，at_ms/every_ms/expr/tz是存储模型；工具面向模型的参数又是另一层。


<a id="section-10-5"></a>

## 10.5 Cron工具参数

源码：[nanobot/agent/tools/cron.py:131](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/cron.py#L131)。

流程伪代码：

```text
CronTool.execute(action, ...):
    读取动作与时间参数
    对新增动作，将 at/every_seconds/cron_expr 等转为 CronSchedule
    委托 CronService 管理任务
    返回操作结果
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
