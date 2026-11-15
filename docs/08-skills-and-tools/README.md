# 08 - 技能与工具

> 已按本地 nanobot `0.4.1`、提交 `27f1a836b5c04a30763dd91960059f81c05bf505` 核对。核心源码使用中文 # 注释解读，配置示例单独标注。详见 [版本核对记录](../LOCAL_SOURCE_BASELINE.md)。

保留Skill概念与独立学习示例，工具参数、注册、来源覆盖、MCP和安全机制以当前源码为准。静态清单不代表所有工具默认启用。

## 目录

- [8.1 Skills 系统概述](#section-8-1)
- [8.2 正确的技能元数据](#section-8-2)
- [8.3 发现、覆盖与渐进披露](#section-8-3)
- [8.4 当前工具定位表](#section-8-4)
- [8.5 ToolRegistry与参数](#section-8-5)
- [8.6 文件、Shell与搜索](#section-8-6)
- [8.7 MCP、子任务与Cron](#section-8-7)
- [8.8 安全与验证](#section-8-8)
- [8.9 独立Skill练习（保留适用示例）](#section-8-9)

<a id="section-8-1"></a>

## 8.1 Skills 系统概述

### 8.1.1 什么是 Skill？

在 Nanobot 中，**Skill（技能）** 是一种可插拔的能力扩展机制。你可以把它理解为"给 Agent 添加的专业模块"：

```
Agent 基础能力            Agent + Skills
┌──────────────┐        ┌──────────────┐
│ · 对话       │        │ · 对话       │
│ · 文件操作   │   →    │ · 文件操作   │
│ · Shell命令  │        │ · Shell命令  │
└──────────────┘        │ · GitHub管理 │  ← Skill
                        │ · 天气查询   │  ← Skill
                        │ · 内容总结   │  ← Skill
                        │ · 代码部署   │  ← Skill
                        └──────────────┘
```

### 8.1.2 Skill vs Tool 的区别

这是面试中容易混淆的概念：

| 维度 | Skill（技能） | Tool（工具） |
|------|-------------|-------------|
| 粒度 | 粗粒度（一个完整能力） | 细粒度（一个具体操作） |
| 定义方式 | Markdown 文件（SKILL.md） | Python 代码注册 |
| 包含内容 | 指令 + 脚本 + 参考资料 | 函数签名 + 执行逻辑 |
| 注入方式 | System Prompt | Tool Definition |
| 类比 | 一门专业课程 | 一个具体工具 |

**关系**：一个 Skill 可以**教会** Agent 如何更好地使用多个 Tool。

```
Skill: GitHub 管理
├── 知道如何创建 PR（指令）
├── 知道代码审查流程（知识）
└── 会使用以下 Tools：
    ├── exec（运行 git 命令）
    ├── read_file（读取代码）
    └── web_fetch（查看 GitHub API）
```


<a id="section-8-2"></a>

## 8.2 正确的技能元数据

下面是独立教学技能示例，不是上游GitHub技能原文。目录名须与name一致，满足名称/描述约束：

```markdown
---
name: learning-reader
description: Read learning notes and summarize only facts supported by the file.
metadata: {"nanobot":{"requires":{"bins":["python"]}}}
---
# 学习记录阅读
先确认当前项目，只读取练习文件；区分文件事实与自己的推断。
```

metadata采用nanobot嵌套requires结构，不是nanobot.requires.bins字面键。具体安装/依赖、always和可用性以当前解析器为准。

源码：[nanobot/agent/skill_metadata.py:32](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/skill_metadata.py#L32)，`valid_skill_metadata`。

```python
def valid_skill_metadata(metadata: dict[str, object], name: str) -> bool:
    """Return whether metadata satisfies the Agent Skills identity contract."""
    # 先取描述，再核对技能身份、格式与长度约束。
    description = metadata.get("description")
    return (
        metadata.get("name") == name
        and len(name) <= 64
        and SKILL_NAME.fullmatch(name) is not None
        and isinstance(description, str)
        and 1 <= len(description.strip()) <= 1024
    )
```

**关键点解读：**名称必须与目录身份一致，且名称和描述有长度、格式约束。这里校验的是 Skill 身份元数据，依赖是否安装与正文是否合理需要另外判断。条件用 and 连接，任一条件不满足都会拒绝该元数据。


<a id="section-8-3"></a>

## 8.3 发现、覆盖与渐进披露

SkillsLoader发现工作区skills，再加入启用插件技能，最后补内置同名未覆盖项；还应用禁用项、别名和依赖检查。snapshot复用本次发现与元数据，不能用“内置永远覆盖工作区”解释。

ContextBuilder加载always技能内容与其他技能摘要；显式$技能入口也有上下文逻辑。不是所有Skill每轮始终全文注入。

源码：[nanobot/agent/skills.py:147](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/skills.py#L147)

```python
# 内置技能最后补入，前面已登记的同名项不会被覆盖。
if self.builtin_skills and self.builtin_skills.exists():
    skills.extend(
        # seen_names 保留工作区与插件的来源优先级。
        self._skill_entries_from_dir(self.builtin_skills, "builtin", skip_names=seen_names)
    )

# 用户禁用列表最终过滤发现结果。
if disabled:
    skills = [s for s in skills if s["name"] not in disabled]

# 返回发现清单；依赖与加载条件还需沿调用方理解。
return skills
```

**关键点解读：**

1. `seen_names` 解决同名来源的优先级，不是按文件修改时间选择“最新”。
2. 工作区技能先进入结果，插件再补充，内置最后补充；所以工作区可以覆盖同名内置技能。
3. 发现与实际可用性检查要区分：后续还会核对依赖与元数据。
4. snapshot 让一次上下文组装使用一致的发现结果；always 技能加载全文，其余技能先展示摘要。


<a id="section-8-4"></a>

## 8.4 当前工具定位表

下表从本地工具类name属性的固定字符串提取，是静态定位表，不保证每个实例全部启用；动态MCP、插件和运行时过滤还会改变清单。实际模型看到的定义由ToolRegistry导出。

| 工具名 | 实现 |
| --- | --- |
| `apply_patch` | [nanobot/agent/tools/apply_patch.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/apply_patch.py)：`ApplyPatchTool` |
| `run_cli_app` | [nanobot/agent/tools/cli_apps.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/cli_apps.py)：`CliAppsTool` |
| `cron` | [nanobot/agent/tools/cron.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/cron.py)：`CronTool` |
| `exec_session` | [nanobot/agent/tools/exec_session.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/exec_session.py)：`ExecSessionTool` |
| `list_exec_sessions` | [nanobot/agent/tools/exec_session.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/exec_session.py)：`ListExecSessionsTool` |
| `read_file` | [nanobot/agent/tools/filesystem.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/filesystem.py)：`ReadFileTool` |
| `write_file` | [nanobot/agent/tools/filesystem.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/filesystem.py)：`WriteFileTool` |
| `edit_file` | [nanobot/agent/tools/filesystem.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/filesystem.py)：`EditFileTool` |
| `list_dir` | [nanobot/agent/tools/filesystem.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/filesystem.py)：`ListDirTool` |
| `generate_image` | [nanobot/agent/tools/image_generation.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/image_generation.py)：`ImageGenerationTool` |
| `message` | [nanobot/agent/tools/message.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/message.py)：`MessageTool` |
| `rg` | [nanobot/agent/tools/rg.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/rg.py)：`RgTool` |
| `find_files` | [nanobot/agent/tools/search.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/search.py)：`FindFilesTool` |
| `grep` | [nanobot/agent/tools/search.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/search.py)：`GrepTool` |
| `my` | [nanobot/agent/tools/self.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/self.py)：`MyTool` |
| `send_session_message` | [nanobot/agent/tools/session_messages.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/session_messages.py)：`SendSessionMessageTool` |
| `search_sessions` | [nanobot/agent/tools/sessions.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/sessions.py)：`SearchSessionsTool` |
| `read_session` | [nanobot/agent/tools/sessions.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/sessions.py)：`ReadSessionTool` |
| `exec` | [nanobot/agent/tools/shell.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/shell.py)：`ExecTool` |
| `subagent` | [nanobot/agent/tools/subagent.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/subagent.py)：`SubagentTool` |


<a id="section-8-5"></a>

## 8.5 ToolRegistry与参数

源码：[nanobot/agent/tools/registry.py:40](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/registry.py#L40)，`ToolRegistry.register`。

```python
def register(self, tool: Tool) -> None:
    """Register a tool."""
    # 按工具名称登记实例；注册不等于执行。
    self._tools[tool.name] = tool
    # 注册表变化后让模型工具定义缓存失效。
    self._cached_definitions = None
```

**关键点解读：**`tool.name` 是索引键，值是实例；注册后清空定义缓存，使下次导出工具定义时反映当前注册表。它没有直接执行工具，正好体现发现、注册与执行的区别。

源码：[nanobot/agent/tools/registry.py:151](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/registry.py#L151)

```python
# 按 Schema 转换参数后进行值与结构校验。
cast_params = tool.cast_params(cast(dict[str, Any], params))
errors = tool.validate_params(cast_params)
# 校验不通过就返回错误，不能直接执行。
if errors:
    return tool, cast_params, (
        ToolResult.error(f"Error: Invalid parameters for tool '{name}': " + "; ".join(errors))
    )
# 成功时返回工具、转换后参数以及空错误。
return tool, cast_params, None
```

**关键点解读：**

1. 名称、参数格式、参数值依次检查；模型输出的工具调用需要经过这些检查才能执行。
2. 返回三项是为了把“找到的实例”“转换后的参数”“失败原因”一起交给执行入口。
3. 例如 `read_file` 的路径字段应符合该工具声明；提供一个工具名和任意字典不会绕过校验。
4. 这个方法准备调用，不执行文件操作。真正副作用发生在后续 `tool.execute(**params)`。

register接收Tool对象。Schema指导类型转换并校验，未知参数/名称不能默认当成任意Python调用。工具execute的实参不从旧教程的手写“parameters说明字典”推断。


<a id="section-8-6"></a>

## 8.6 文件、Shell与搜索

read_file等参数以当前装饰器Schema/execute签名为准，不能沿用start_line/end_line之类未经核对的字段。Shell通过ExecTool和会话管理处理进程、超时、输出、取消与沙箱；不是只设置cwd就安全。网页搜索配置位于tools.web.search，不是tools.web_search。

当前 config.json **局部配置示例**：合并到已有配置，不覆盖整份文件；占位参数需替换。

```json
{
  "tools": {
    "web": {
      "enable": true,
      "search": {
        "provider": "duckduckgo",
        "maxResults": 5,
        "timeout": 30
      }
    }
  }
}
```


<a id="section-8-7"></a>

## 8.7 MCP、子任务与Cron

MCPProvider管理外部连接；Wrapper.execute返回结果。subagent工具action为create/send/cancel/check，Cron工具直接接收every_seconds、cron_expr、tz、at等字段，不是统一嵌套schedule对象。

独立Skill或业务脚本只是示例，保存到自己的练习/Agent技能目录；本次仅核对文档，不宣称已经部署。


<a id="section-8-8"></a>

## 8.8 安全与验证

输入Schema、内容可信度、工作区范围和OS沙箱是不同边界。read_only/concurrency_safe/exclusive影响工具调度，不能据工具名称自行猜。网络拒绝不通过换curl等方式绕过。

读test_tool_registry、test_skills_loader、test_runner_tool_execution，真实文件操作只用练习目录。自查：描述如何进入模型、工具如何实际执行、错误如何回传、为什么取消不回滚所有副作用。


<a id="section-8-9"></a>

## 8.9 独立Skill练习（保留适用示例）

这是保留的**独立教学技能/脚本示例**，不是nanobot内部源码；技能应放在实际Agent工作区skills/，当前元数据和授权以前文为准，运行结果由读者验证。

### 8.10.1 实战：创建一个"代码分析"技能

**Step 1：创建技能目录**

```bash
mkdir -p skills/code-analyzer
```

**Step 2：编写 SKILL.md**

```markdown
---
name: code-analyzer
description: "Python 代码质量分析与优化建议"
always: false
metadata: '{"nanobot.requires.bins": ["python3"]}'
---

# 代码分析师技能

你是一名资深的 Python 代码分析师。当用户请求代码分析时，按以下流程操作。

## 分析维度

1. **代码规范**：PEP 8 合规性、命名规范
2. **安全性**：SQL 注入、路径遍历、硬编码密钥
3. **性能**：算法复杂度、不必要的计算、N+1 查询
4. **可维护性**：函数长度、圈复杂度、耦合度
5. **测试覆盖**：关键路径是否有测试

## 分析流程

### Step 1: 收集代码

```
使用 list_dir 查看项目结构 使用 read_file 读取关键文件
```

### Step 2: 运行静态分析

```
使用 exec 运行以下命令（如果工具可用）：
- python3 -m py_compile <file>  # 语法检查
- python3 -m pylint <file>      # 代码质量
- python3 -m bandit <file>      # 安全扫描
```

### Step 3: 生成报告

按以下格式生成分析报告：

## 报告模板

### 📊 总体评分: X/10

### 🔴 严重问题
(列出所有严重问题)

### 🟡 改进建议
(列出所有改进建议)

### 🟢 优秀实践
(列出代码中做得好的地方)

### 📝 重构建议
(给出具体的重构方案)
```

**Step 3：添加参考资料（可选）**

```bash
mkdir -p skills/code-analyzer/references
```

```markdown
<!-- skills/code-analyzer/references/security-checklist.md -->
# Python 安全检查清单

## 常见安全漏洞

1. SQL 注入：使用参数化查询而非字符串拼接
2. 路径遍历：验证用户输入的文件路径
3. 命令注入：避免 os.system()，使用 subprocess
4. 硬编码密钥：使用环境变量或密钥管理服务
5. 不安全的反序列化：避免 pickle.loads() 处理不信任数据
```

**Step 4：添加辅助脚本（可选）**

```bash
mkdir -p skills/code-analyzer/scripts
```

```python
#!/usr/bin/env python3
# skills/code-analyzer/scripts/complexity.py
"""计算 Python 文件的圈复杂度"""
import ast
import sys

def calculate_complexity(filepath):
    with open(filepath) as f:
        tree = ast.parse(f.read())

    for node in ast.walk(tree):
        if isinstance(node, (ast.FunctionDef, ast.AsyncFunctionDef)):
            complexity = 1
            for child in ast.walk(node):
                if isinstance(child, (ast.If, ast.While, ast.For,
                                     ast.ExceptHandler, ast.With,
                                     ast.BoolOp)):
                    complexity += 1
            print(f"{node.name}: complexity = {complexity}")

if __name__ == "__main__":
    calculate_complexity(sys.argv[1])
```

**Step 5：测试技能**

```bash
# 启动 Agent
nanobot

# 测试对话：
# You: 请分析一下 src/main.py 的代码质量
# Agent 应该会读取 SKILL.md，然后按照分析流程执行
```

### 8.10.2 实战：创建一个"面试教练"技能

```markdown
---
name: interview-coach
description: "AI Agent 方向的面试训练和评估"
always: false
---

# 面试教练技能

## 模式

### 模式 1: 知识测验
随机从知识库中抽取问题，评估用户的回答。

### 模式 2: 模拟面试
模拟真实面试场景，包括追问和压力测试。

### 模式 3: 答案优化
帮助用户优化已有的面试回答。

## 评分标准

- 技术准确性 (40%)
- 表达清晰度 (20%)
- 深度和广度 (20%)
- 实际经验体现 (20%)

## 知识库主题

1. AI Agent 基础概念
2. 框架架构设计
3. 记忆系统
4. 工具与技能系统
5. 多平台部署
6. 安全与生产环境

## 反馈格式

### 评分: X/10

**优点**：
- ...

**不足**：
- ...

**改进建议**：
- ...

**参考答案**：
(给出一个更完善的回答示例)
```

## 下一章

[继续第 09 章](../09-multi-platform/README.md)
