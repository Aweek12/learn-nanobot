# 06 - 安装与上手

> 已按本地 nanobot `0.4.1`、提交 `27f1a836b5c04a30763dd91960059f81c05bf505` 核对。方法使用本地新版完整源码并配中文 # 注释解读，配置示例单独标注。详见 [版本核对记录](../LOCAL_SOURCE_BASELINE.md)。

以本地源码安装和WebUI模型预设为主线，保留可复制命令，但不覆盖读者现有实例或要求全局放宽执行策略。

## 目录

- [6.1 环境与版本](#section-6-1)
- [6.2 源码安装](#section-6-2)
- [6.3 WebUI配置模型](#section-6-3)
- [6.4 配置字段与默认值](#section-6-4)
- [6.5 目录与引导文件](#section-6-5)
- [6.6 首次对话和文件验证](#section-6-6)
- [6.7 生命周期与排错](#section-6-7)
- [6.8 依据与自查](#section-6-8)

<a id="section-6-1"></a>

## 6.1 环境与版本

Python最低3.11，Windows原生可用，不要求必须WSL。源码客户端需要Git/Bun相关流程。源码检出、PyPI发行包和官方安装器环境可能不同；先确认解释器与nanobot.__file__，不一边读本地代码一边运行另一个发布包。


<a id="section-6-2"></a>

## 6.2 源码安装

已有仓库不要再次嵌套clone。以下是Windows PowerShell示例：

```powershell
cd E:\codexwork\nanobot
git rev-parse HEAD
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -e .
.\.venv\Scripts\python.exe -c "import sys,nanobot; print(sys.executable); print(nanobot.__file__)"
.\.venv\Scripts\python.exe -m nanobot --version
```

已经创建正确环境的读者跳过venv和重装。精确复现需要基准SHA，不用版本字符串替代；切换版本前保留本地工作，不使用强制重置。Bun、依赖或构建失败按实际输出定位。


<a id="section-6-3"></a>

## 6.3 WebUI配置模型

```powershell
.\.venv\Scripts\python.exe -m nanobot webui
```

打开Settings → Models → New preset，选择/添加Provider，填写所需Key、Base URL和准确模型ID，保存后在对话输入区选择预设。已配置成功无需重复填写。完整登录链接含认证片段，不公开分享。

终端配置可用 `nanobot onboard --wizard`；普通 `onboard` 不能不加区分地说成所有版本都运行完整向导。


<a id="section-6-4"></a>

## 6.4 配置字段与默认值

当前 config.json **局部配置示例**：合并到已有配置，不覆盖整份文件；占位参数需替换。

```json
{
  "agents": {
    "defaults": {
      "maxToolIterations": 200,
      "maxConcurrentSubagents": 4,
      "maxToolResultChars": 16000,
      "timezone": "Asia/Shanghai"
    }
  },
  "gateway": {
    "heartbeat": {
      "enabled": true,
      "intervalS": 1800
    }
  }
}
```

这里显式选择时区，不声称所有实例默认北京时间。maxToolResultChars与Token预算不同；主/子执行还受继续与控制策略影响。配置根是config.json，预设通常由modelPresets与agents.defaults.modelPreset引用。


<a id="section-6-5"></a>

## 6.5 目录与引导文件

配置默认在~/.nanobot/config.json，会话在配置目录sessions/<workspace-id>/sessions.sqlite3；Agent工作区有SOUL、USER、memory、skills、HEARTBEAT等。会话可选单独项目，其AGENTS与相对工具路径不等于Agent资料目录。

根仓库AGENTS用于开发，templates/AGENTS是默认模板，用户工作区实际指令另有来源。不要将“身份定义全部放AGENTS”作为唯一机制。


<a id="section-6-6"></a>

## 6.6 首次对话和文件验证

先一句Hello验证当次实际模型路径，再在独立练习目录创建notes.txt，选中该项目，请Agent读取后总结，再新建summary.md并检查磁盘。附件发送不会自动改变项目目录；模型说已完成不代表文件已写入。

status只检查本地配置，不代表已实际调用模型；成功也可能经过备用预设，记录实际来源。


<a id="section-6-7"></a>

## 6.7 生命周期与排错

源码启动器可能加入已有gateway，按需模式与最后启动器退出相关；官方安装器或显式持久模式不同。关闭浏览器、Ctrl+C和停止服务不是同一动作。用gateway status查看对应实例，停止前确认无其他任务。

Python Console只接收Python语句，Terminal接收命令。找不到nanobot时可用明确Python路径的-m入口；误用Anaconda先核对解释器。文件失败先看工作区与授权，不先扩大到全盘访问。


<a id="section-6-8"></a>

## 6.8 依据与自查

[README.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/README.md) · [docs/quick-start.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/quick-start.md) · [docs/webui.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/webui.md) · [pyproject.toml](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/pyproject.toml)

能说明源码路径、配置实例、实际模型、项目目录与文件结果；未完成的真实网络或工具实验不宣称通过。

## 下一章

[继续第 07 章](../07-memory-system/README.md)
