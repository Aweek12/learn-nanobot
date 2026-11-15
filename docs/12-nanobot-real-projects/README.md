# 12 - 应用场景中的当前机制走读

> 已按本地 nanobot `0.4.1`、提交 `27f1a836b5c04a30763dd91960059f81c05bf505` 核对。核心源码使用中文 # 注释解读，配置示例单独标注。详见 [版本核对记录](../LOCAL_SOURCE_BASELINE.md)。

旧章节中与当前框架不兼容的配置、消息路由与启动示例已替换为场景走读及当前SDK/MCP入口；独立业务功能不冒充上游实现。

## 目录

- [12.1 阅读定位与代码边界](#section-12-1)
- [12.2 场景一：运维信息摘要](#section-12-2)
- [12.3 场景二：多渠道客服](#section-12-3)
- [12.4 场景三：只读数据服务](#section-12-4)
- [12.5 从Python复用现有运行时](#section-12-5)
- [12.6 如何验证与描述](#section-12-6)

<a id="section-12-1"></a>

## 12.1 阅读定位与代码边界

保留原章节的运维、客服与数据分析三个场景，但用它们解释现有框架，不把自建消息路由器、简化JSON服务或不存在的nanobot chat命令当成上游实现。

仓库projects/下原示例不在本次docs修订范围，未声称已兼容或实测。需要运行旧示例时先逐项审查依赖、配置与安全；不要从本文推断它们已验证。


<a id="section-12-2"></a>

## 12.2 场景一：运维信息摘要

在独立练习目录放入脱敏日志，选中该项目，先读文件后生成新报告，检查真实输出。定时化由Cron触发已有Agent路径，不需要把APScheduler嵌进Loop。SOUL/Skill是行为说明，实际文件和Shell权限由后端检查。

下面截取 Runner 中准备上下文和请求模型的真实语句，用 # 注释说明每一步。

1. 选择已配置模型和练习项目
2. 读取脱敏日志文件，核对工具结果
3. 模型总结并标明证据与未知项
4. 写入新报告，检查实际文件
5. 如需周期执行，显式配置Cron并核对通知结果


<a id="section-12-3"></a>

## 12.3 场景二：多渠道客服

渠道将平台消息映射为InboundMessage，manager与消息总线交付输出；不用另外HTTP /chat路由器假装当前Python SDK。身份与session namespace需要明确，跨平台统一画像是应用设计，不是框架自动保证。

配置由WebUI Channels或当前包的配置契约提供；保留已有模型预设、渠道与工具配置，不覆盖整份config。知识库检索若由外部MCP提供，其返回文本是数据而非新的权限指令。


<a id="section-12-4"></a>

## 12.4 场景三：只读数据服务

MCPProvider连接服务、发现能力，Wrapper将调用映射到本地Tool。数据库只读账号、输入/查询限制和结果预算要在服务端真正实施，不能把“只读SQL”Prompt当成数据库授权。

当前 config.json **局部配置示例**：合并到已有配置，不覆盖整份文件；占位参数需替换。

```json
{
  "tools": {
    "mcpServers": {
      "database": {
        "command": "E:/tools/python/python.exe",
        "args": [
          "E:/my-services/db_server.py"
        ],
        "enabledTools": [
          "query_database",
          "list_tables"
        ],
        "toolTimeout": 30
      }
    }
  }
}
```

可执行路径、脚本与工具名是待替换示例，不是仓库已包含的服务。只保留确实存在且需要的能力，未启动、连接、调用的服务明确记为未验证。


<a id="section-12-5"></a>

## 12.5 从Python复用现有运行时

下面是当前官方SDK风格的**独立使用示例**，不是nanobot内部源码。from_config使用现有实例配置，可能调用真实模型和工具；本次不自动执行：

```python
import asyncio
from nanobot import Nanobot

async def main():
    async with Nanobot.from_config() as bot:
        result = await bot.run("请用一句话介绍当前任务，不需要操作文件。")
    print(result.content)

if __name__ == "__main__":
    asyncio.run(main())
```

Prompt要求不操作不等于后端禁用权限。参数与结果以sdk/types、clients和Nanobot公共接口为准。

[docs/python-sdk.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/python-sdk.md)


<a id="section-12-6"></a>

## 12.6 如何验证与描述

先验证普通模型响应，再验证工具/路径，再验证存储和交付；Mock测试与真实部署分别记录。展示实际消息、工具ID、停止原因和文件结果，不填未经测量的吞吐、成本或成功率。

面试区分上游原有能力、自己的阅读验证与真正完成的独立应用；没有开发过的业务服务不能据此写成个人项目成果。

## 下一章

[继续第 13 章](../13-interview-bagua/README.md)

## 附录：保留的独立业务代码

以下内容与当前nanobot内部实现分开，未作为可复制即生产的方案。运行前需独立核对服务协议、依赖、凭据和权限；projects目录仍未修改。

### ops_alert.py - 运维告警通知模块

**原教程独立业务示例**，不是nanobot源码，也不代表已验证当前MCP握手、认证、只读安全或服务部署；本次仅保留其业务逻辑供阅读。协议接入必须使用本章的当前MCPProvider/SDK契约，不能把下列自定义JSON循环作为完整MCP实现。

```python
#!/usr/bin/env python3
"""
ops_alert.py - 运维告警通知模块
支持飞书和钉钉 Webhook 通知
"""

import json
import os
import urllib.request
from datetime import datetime
from typing import Optional


class AlertLevel:
    INFO = "info"
    WARNING = "warning"
    CRITICAL = "critical"


class OpsAlertNotifier:
    """运维告警通知器，支持飞书和钉钉"""

    def __init__(self):
        self.feishu_webhook = os.getenv("FEISHU_WEBHOOK_URL")
        self.dingtalk_webhook = os.getenv("DINGTALK_WEBHOOK_URL")

    def send_alert(
        self,
        title: str,
        content: str,
        level: str = AlertLevel.INFO,
        server: str = "unknown",
    ):
        """发送告警到所有已配置的通道"""
        results = {}

        if self.feishu_webhook:
            results["feishu"] = self._send_feishu(title, content, level, server)

        if self.dingtalk_webhook:
            results["dingtalk"] = self._send_dingtalk(title, content, level, server)

        return results

    def _send_feishu(
        self, title: str, content: str, level: str, server: str
    ) -> bool:
        """发送飞书通知"""
        level_emoji = {"info": "ℹ️", "warning": "⚠️", "critical": "🔴"}
        emoji = level_emoji.get(level, "ℹ️")

        payload = {
            "msg_type": "interactive",
            "card": {
                "header": {
                    "title": {"tag": "plain_text", "content": f"{emoji} {title}"},
                    "template": {
                        "info": "blue",
                        "warning": "yellow",
                        "critical": "red",
                    }.get(level, "blue"),
                },
                "elements": [
                    {
                        "tag": "div",
                        "text": {
                            "tag": "lark_md",
                            "content": (
                                f"**服务器**: {server}\n"
                                f"**时间**: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n"
                                f"**详情**:\n{content}"
                            ),
                        },
                    }
                ],
            },
        }

        return self._post_webhook(self.feishu_webhook, payload)

    def _send_dingtalk(
        self, title: str, content: str, level: str, server: str
    ) -> bool:
        """发送钉钉通知"""
        level_text = {"info": "提示", "warning": "警告", "critical": "严重"}

        payload = {
            "msgtype": "markdown",
            "markdown": {
                "title": f"[{level_text.get(level, '提示')}] {title}",
                "text": (
                    f"## {title}\n\n"
                    f"- **级别**: {level_text.get(level, '提示')}\n"
                    f"- **服务器**: {server}\n"
                    f"- **时间**: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}\n\n"
                    f"### 详情\n\n{content}"
                ),
            },
        }

        return self._post_webhook(self.dingtalk_webhook, payload)

    def _post_webhook(self, url: str, payload: dict) -> bool:
        """发送 HTTP POST 请求到 Webhook"""
        try:
            data = json.dumps(payload).encode("utf-8")
            req = urllib.request.Request(
                url,
                data=data,
                headers={"Content-Type": "application/json"},
            )
            with urllib.request.urlopen(req, timeout=10) as resp:
                return resp.status == 200
        except Exception as e:
            print(f"告警发送失败: {e}")
            return False


def send_inspection_alert(report: dict):
    """巡检结果告警入口"""
    notifier = OpsAlertNotifier()

    if report.get("overall_status") == "critical":
        notifier.send_alert(
            title="服务器巡检异常 - 需立即处理",
            content=report.get("summary", ""),
            level=AlertLevel.CRITICAL,
            server=report.get("server", "unknown"),
        )
    elif report.get("overall_status") == "warning":
        notifier.send_alert(
            title="服务器巡检警告",
            content=report.get("summary", ""),
            level=AlertLevel.WARNING,
            server=report.get("server", "unknown"),
        )


if __name__ == "__main__":
    send_inspection_alert({
        "overall_status": "warning",
        "server": "prod-web-01",
        "summary": "CPU 使用率 82%，超过警告阈值 80%。\n建议检查高 CPU 进程。",
    })
```

### ticket_server.py - 工单管理 MCP Server

**原教程独立业务示例**，不是nanobot源码，也不代表已验证当前MCP握手、认证、只读安全或服务部署；本次仅保留其业务逻辑供阅读。协议接入必须使用本章的当前MCPProvider/SDK契约，不能把下列自定义JSON循环作为完整MCP实现。

```python
#!/usr/bin/env python3
"""
ticket_server.py - 工单管理 MCP Server
基于 MCP 协议实现的工单 CRUD 系统，使用 SQLite 存储
"""

import json
import sqlite3
import sys
import uuid
from datetime import datetime
from typing import Any


DB_PATH = "tickets.db"


def init_database():
    """初始化数据库表结构"""
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS tickets (
            ticket_id TEXT PRIMARY KEY,
            title TEXT NOT NULL,
            description TEXT NOT NULL,
            priority TEXT DEFAULT 'medium',
            status TEXT DEFAULT 'open',
            user_id TEXT NOT NULL,
            user_name TEXT DEFAULT '',
            platform TEXT DEFAULT '',
            created_at TEXT NOT NULL,
            updated_at TEXT NOT NULL,
            comments TEXT DEFAULT '[]'
        )
    """)
    conn.commit()
    conn.close()


class TicketMCPServer:
    """工单管理 MCP Server"""

    def __init__(self):
        init_database()

    def handle_request(self, request: dict) -> dict:
        """处理 MCP 请求"""
        method = request.get("method", "")
        req_id = request.get("id")

        handlers = {
            "initialize": self._handle_initialize,
            "tools/list": self._handle_list_tools,
            "tools/call": self._handle_call_tool,
        }

        handler = handlers.get(method)
        if handler:
            result = handler(request)
            return {"jsonrpc": "2.0", "id": req_id, "result": result}

        return {
            "jsonrpc": "2.0",
            "id": req_id,
            "error": {"code": -32601, "message": f"Unknown method: {method}"},
        }

    def _handle_initialize(self, request: dict) -> dict:
        return {
            "protocolVersion": "2024-11-05",
            "serverInfo": {
                "name": "ticket-server",
                "version": "1.0.0",
            },
            "capabilities": {"tools": {}},
        }

    def _handle_list_tools(self, request: dict) -> dict:
        return {
            "tools": [
                {
                    "name": "create_ticket",
                    "description": "创建新的客服工单",
                    "inputSchema": {
                        "type": "object",
                        "properties": {
                            "title": {
                                "type": "string",
                                "description": "工单标题",
                            },
                            "description": {
                                "type": "string",
                                "description": "问题详细描述",
                            },
                            "priority": {
                                "type": "string",
                                "enum": ["low", "medium", "high", "urgent"],
                                "description": "优先级",
                                "default": "medium",
                            },
                            "user_id": {
                                "type": "string",
                                "description": "用户 ID",
                            },
                            "user_name": {
                                "type": "string",
                                "description": "用户名称",
                            },
                            "platform": {
                                "type": "string",
                                "description": "来源平台",
                            },
                        },
                        "required": ["title", "description", "user_id"],
                    },
                },
                {
                    "name": "get_ticket",
                    "description": "根据工单 ID 查询工单详情",
                    "inputSchema": {
                        "type": "object",
                        "properties": {
                            "ticket_id": {
                                "type": "string",
                                "description": "工单 ID",
                            },
                        },
                        "required": ["ticket_id"],
                    },
                },
                {
                    "name": "list_user_tickets",
                    "description": "列出指定用户的所有工单",
                    "inputSchema": {
                        "type": "object",
                        "properties": {
                            "user_id": {
                                "type": "string",
                                "description": "用户 ID",
                            },
                            "status": {
                                "type": "string",
                                "enum": [
                                    "open",
                                    "in_progress",
                                    "resolved",
                                    "closed",
                                ],
                                "description": "按状态过滤",
                            },
                        },
                        "required": ["user_id"],
                    },
                },
                {
                    "name": "update_ticket",
                    "description": "更新工单状态或添加备注",
                    "inputSchema": {
                        "type": "object",
                        "properties": {
                            "ticket_id": {
                                "type": "string",
                                "description": "工单 ID",
                            },
                            "status": {
                                "type": "string",
                                "enum": [
                                    "open",
                                    "in_progress",
                                    "resolved",
                                    "closed",
                                ],
                                "description": "新状态",
                            },
                            "comment": {
                                "type": "string",
                                "description": "更新备注",
                            },
                        },
                        "required": ["ticket_id"],
                    },
                },
            ]
        }

    def _handle_call_tool(self, request: dict) -> dict:
        params = request.get("params", {})
        tool_name = params.get("name", "")
        arguments = params.get("arguments", {})

        tool_handlers = {
            "create_ticket": self._create_ticket,
            "get_ticket": self._get_ticket,
            "list_user_tickets": self._list_user_tickets,
            "update_ticket": self._update_ticket,
        }

        handler = tool_handlers.get(tool_name)
        if not handler:
            return {
                "content": [
                    {
                        "type": "text",
                        "text": f"未知工具: {tool_name}",
                    }
                ],
                "isError": True,
            }

        try:
            result = handler(arguments)
            return {
                "content": [
                    {"type": "text", "text": json.dumps(result, ensure_ascii=False)}
                ]
            }
        except Exception as e:
            return {
                "content": [{"type": "text", "text": f"执行出错: {str(e)}"}],
                "isError": True,
            }

    def _create_ticket(self, args: dict) -> dict:
        """创建工单"""
        ticket_id = f"TK-{uuid.uuid4().hex[:8].upper()}"
        now = datetime.now().isoformat()

        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        cursor.execute(
            """
            INSERT INTO tickets
                (ticket_id, title, description, priority, status,
                 user_id, user_name, platform, created_at, updated_at)
            VALUES (?, ?, ?, ?, 'open', ?, ?, ?, ?, ?)
            """,
            (
                ticket_id,
                args["title"],
                args["description"],
                args.get("priority", "medium"),
                args["user_id"],
                args.get("user_name", ""),
                args.get("platform", ""),
                now,
                now,
            ),
        )
        conn.commit()
        conn.close()

        return {
            "success": True,
            "ticket_id": ticket_id,
            "message": f"工单 {ticket_id} 创建成功",
            "created_at": now,
        }

    def _get_ticket(self, args: dict) -> dict:
        """查询工单"""
        conn = sqlite3.connect(DB_PATH)
        conn.row_factory = sqlite3.Row
        cursor = conn.cursor()
        cursor.execute(
            "SELECT * FROM tickets WHERE ticket_id = ?", (args["ticket_id"],)
        )
        row = cursor.fetchone()
        conn.close()

        if not row:
            return {"success": False, "message": "工单不存在"}

        return {"success": True, "ticket": dict(row)}

    def _list_user_tickets(self, args: dict) -> dict:
        """列出用户工单"""
        conn = sqlite3.connect(DB_PATH)
        conn.row_factory = sqlite3.Row
        cursor = conn.cursor()

        if "status" in args:
            cursor.execute(
                "SELECT * FROM tickets WHERE user_id = ? AND status = ? ORDER BY created_at DESC",
                (args["user_id"], args["status"]),
            )
        else:
            cursor.execute(
                "SELECT * FROM tickets WHERE user_id = ? ORDER BY created_at DESC",
                (args["user_id"],),
            )

        rows = cursor.fetchall()
        conn.close()

        return {
            "success": True,
            "count": len(rows),
            "tickets": [dict(row) for row in rows],
        }

    def _update_ticket(self, args: dict) -> dict:
        """更新工单"""
        conn = sqlite3.connect(DB_PATH)
        conn.row_factory = sqlite3.Row
        cursor = conn.cursor()

        cursor.execute(
            "SELECT * FROM tickets WHERE ticket_id = ?", (args["ticket_id"],)
        )
        row = cursor.fetchone()
        if not row:
            conn.close()
            return {"success": False, "message": "工单不存在"}

        now = datetime.now().isoformat()
        updates = []
        values = []

        if "status" in args:
            updates.append("status = ?")
            values.append(args["status"])

        if "comment" in args:
            comments = json.loads(row["comments"])
            comments.append({
                "text": args["comment"],
                "timestamp": now,
            })
            updates.append("comments = ?")
            values.append(json.dumps(comments, ensure_ascii=False))

        updates.append("updated_at = ?")
        values.append(now)
        values.append(args["ticket_id"])

        cursor.execute(
            f"UPDATE tickets SET {', '.join(updates)} WHERE ticket_id = ?",
            values,
        )
        conn.commit()
        conn.close()

        return {
            "success": True,
            "message": f"工单 {args['ticket_id']} 更新成功",
            "updated_at": now,
        }

    def run(self):
        """启动 MCP Server，通过 stdin/stdout 通信"""
        while True:
            try:
                line = sys.stdin.readline()
                if not line:
                    break

                request = json.loads(line.strip())
                response = self.handle_request(request)
                sys.stdout.write(json.dumps(response) + "\n")
                sys.stdout.flush()
            except json.JSONDecodeError:
                continue
            except KeyboardInterrupt:
                break


if __name__ == "__main__":
    server = TicketMCPServer()
    server.run()
```

### db_server.py - 安全的数据库查询 MCP Server

**原教程独立业务示例**，不是nanobot源码，也不代表已验证当前MCP握手、认证、只读安全或服务部署；本次仅保留其业务逻辑供阅读。协议接入必须使用本章的当前MCPProvider/SDK契约，不能把下列自定义JSON循环作为完整MCP实现。

```python
#!/usr/bin/env python3
"""
db_server.py - 安全的数据库查询 MCP Server

提供四个工具：
- query_database: 执行只读 SQL 查询
- list_tables: 列出所有数据表
- describe_table: 查看表结构
- get_statistics: 获取表的统计信息

安全特性：
- 只允许 SELECT 语句
- SQL 注入检测
- 查询超时控制
- 结果行数限制
- 完整的审计日志
"""

import json
import os
import re
import signal
import sqlite3
import sys
import threading
import time
from contextlib import contextmanager
from datetime import datetime
from typing import Any, Optional


# ==================== 配置 ====================

class Config:
    DB_PATH = os.getenv("DB_PATH", "business.db")
    QUERY_TIMEOUT = int(os.getenv("QUERY_TIMEOUT", "30"))
    MAX_ROWS = int(os.getenv("MAX_ROWS", "1000"))
    MAX_CONCURRENT = int(os.getenv("MAX_CONCURRENT", "5"))
    AUDIT_LOG_PATH = os.getenv("AUDIT_LOG_PATH", "audit.log")
    READONLY = os.getenv("DB_READONLY", "true").lower() == "true"


# ==================== 审计日志 ====================

class AuditLogger:
    """查询审计日志记录器"""

    def __init__(self, log_path: str):
        self.log_path = log_path
        self._lock = threading.Lock()

    def log_query(
        self,
        sql: str,
        success: bool,
        duration_ms: float,
        rows_returned: int = 0,
        error: str = "",
    ):
        entry = {
            "timestamp": datetime.now().isoformat(),
            "type": "query",
            "sql": sql,
            "success": success,
            "duration_ms": round(duration_ms, 2),
            "rows_returned": rows_returned,
            "error": error,
        }
        self._write(entry)

    def log_blocked(self, sql: str, reason: str):
        entry = {
            "timestamp": datetime.now().isoformat(),
            "type": "blocked",
            "sql": sql,
            "reason": reason,
        }
        self._write(entry)

    def _write(self, entry: dict):
        with self._lock:
            try:
                with open(self.log_path, "a", encoding="utf-8") as f:
                    f.write(json.dumps(entry, ensure_ascii=False) + "\n")
            except IOError:
                pass


# ==================== SQL 安全检查 ====================

class SQLSecurityChecker:
    """SQL 安全检查器 - 多层防护"""

    DANGEROUS_KEYWORDS = [
        "DROP", "DELETE", "UPDATE", "INSERT", "ALTER", "TRUNCATE",
        "CREATE", "REPLACE", "GRANT", "REVOKE", "EXEC", "EXECUTE",
        "CALL", "MERGE", "UPSERT", "ATTACH", "DETACH",
    ]

    INJECTION_PATTERNS = [
        r";\s*(DROP|DELETE|UPDATE|INSERT|ALTER)",
        r"UNION\s+(ALL\s+)?SELECT",
        r"OR\s+1\s*=\s*1",
        r"OR\s+'1'\s*=\s*'1'",
        r"--\s*$",
        r"/\*.*\*/",
        r"SLEEP\s*\(",
        r"BENCHMARK\s*\(",
        r"LOAD_FILE\s*\(",
        r"INTO\s+(OUTFILE|DUMPFILE)",
        r"@@(version|datadir|basedir)",
        r"INFORMATION_SCHEMA",
    ]

    def check(self, sql: str) -> tuple[bool, str]:
        """
        检查 SQL 是否安全。
        返回 (is_safe, reason)
        """
        if not sql or not sql.strip():
            return False, "空查询"

        normalized = sql.strip().upper()

        if not self._is_select_statement(normalized):
            return False, "只允许 SELECT 查询语句"

        keyword_check = self._check_dangerous_keywords(normalized)
        if not keyword_check[0]:
            return keyword_check

        injection_check = self._check_injection_patterns(sql)
        if not injection_check[0]:
            return injection_check

        if normalized.count(";") > 0:
            parts = [p.strip() for p in normalized.split(";") if p.strip()]
            if len(parts) > 1:
                return False, "不允许多语句执行"

        return True, "通过安全检查"

    def _is_select_statement(self, sql: str) -> bool:
        cleaned = sql.lstrip("( \t\n")
        return cleaned.startswith("SELECT") or cleaned.startswith("WITH")

    def _check_dangerous_keywords(self, sql: str) -> tuple[bool, str]:
        words = set(re.findall(r'\b[A-Z_]+\b', sql))
        for keyword in self.DANGEROUS_KEYWORDS:
            if keyword in words:
                return False, f"包含禁止的关键字: {keyword}"
        return True, ""

    def _check_injection_patterns(self, sql: str) -> tuple[bool, str]:
        for pattern in self.INJECTION_PATTERNS:
            if re.search(pattern, sql, re.IGNORECASE):
                return False, f"检测到可疑的注入模式"
        return True, ""


# ==================== 数据库连接管理 ====================

class DatabaseManager:
    """数据库连接和查询管理器"""

    def __init__(self, db_path: str, readonly: bool = True):
        self.db_path = db_path
        self.readonly = readonly
        self._semaphore = threading.Semaphore(Config.MAX_CONCURRENT)

    @contextmanager
    def get_connection(self):
        """获取数据库连接（带并发控制）"""
        acquired = self._semaphore.acquire(timeout=10)
        if not acquired:
            raise RuntimeError("并发查询数已达上限，请稍后重试")

        try:
            uri = f"file:{self.db_path}?mode=ro" if self.readonly else self.db_path
            conn = sqlite3.connect(
                uri if self.readonly else self.db_path,
                uri=self.readonly,
                timeout=Config.QUERY_TIMEOUT,
            )
            conn.row_factory = sqlite3.Row
            conn.execute(f"PRAGMA busy_timeout = {Config.QUERY_TIMEOUT * 1000}")
            yield conn
        finally:
            conn.close()
            self._semaphore.release()

    def execute_query(self, sql: str, max_rows: int = None) -> dict:
        """执行查询并返回结果"""
        if max_rows is None:
            max_rows = Config.MAX_ROWS

        start_time = time.time()

        with self.get_connection() as conn:
            cursor = conn.cursor()

            timer = threading.Timer(Config.QUERY_TIMEOUT, self._cancel_query, [conn])
            timer.start()

            try:
                cursor.execute(sql)
                rows = cursor.fetchmany(max_rows + 1)
                truncated = len(rows) > max_rows
                if truncated:
                    rows = rows[:max_rows]

                columns = (
                    [description[0] for description in cursor.description]
                    if cursor.description
                    else []
                )

                duration_ms = (time.time() - start_time) * 1000

                return {
                    "success": True,
                    "columns": columns,
                    "rows": [dict(row) for row in rows],
                    "row_count": len(rows),
                    "truncated": truncated,
                    "max_rows": max_rows,
                    "duration_ms": round(duration_ms, 2),
                }
            except sqlite3.OperationalError as e:
                if "interrupted" in str(e).lower():
                    raise TimeoutError("查询超时")
                raise
            finally:
                timer.cancel()

    def list_tables(self) -> list[dict]:
        """列出所有表"""
        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute(
                "SELECT name, type FROM sqlite_master "
                "WHERE type IN ('table', 'view') AND name NOT LIKE 'sqlite_%' "
                "ORDER BY type, name"
            )
            tables = []
            for row in cursor.fetchall():
                count_cursor = conn.cursor()
                try:
                    count_cursor.execute(f'SELECT COUNT(*) FROM "{row["name"]}"')
                    count = count_cursor.fetchone()[0]
                except Exception:
                    count = -1

                tables.append({
                    "name": row["name"],
                    "type": row["type"],
                    "row_count": count,
                })
            return tables

    def describe_table(self, table_name: str) -> dict:
        """查看表结构"""
        if not re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', table_name):
            raise ValueError("无效的表名")

        with self.get_connection() as conn:
            cursor = conn.cursor()
            cursor.execute(f'PRAGMA table_info("{table_name}")')
            columns = []
            for row in cursor.fetchall():
                columns.append({
                    "name": row["name"],
                    "type": row["type"],
                    "nullable": not row["notnull"],
                    "default": row["dflt_value"],
                    "primary_key": bool(row["pk"]),
                })

            cursor.execute(f'PRAGMA index_list("{table_name}")')
            indexes = []
            for row in cursor.fetchall():
                indexes.append({
                    "name": row["name"],
                    "unique": bool(row["unique"]),
                })

            return {
                "table_name": table_name,
                "columns": columns,
                "indexes": indexes,
                "column_count": len(columns),
            }

    def get_statistics(self, table_name: str) -> dict:
        """获取表的统计信息"""
        if not re.match(r'^[a-zA-Z_][a-zA-Z0-9_]*$', table_name):
            raise ValueError("无效的表名")

        with self.get_connection() as conn:
            cursor = conn.cursor()

            cursor.execute(f'SELECT COUNT(*) as total FROM "{table_name}"')
            total = cursor.fetchone()[0]

            cursor.execute(f'PRAGMA table_info("{table_name}")')
            columns_info = cursor.fetchall()

            column_stats = []
            for col in columns_info:
                col_name = col["name"]
                col_type = col["type"].upper()
                stats = {"name": col_name, "type": col["type"]}

                try:
                    cursor.execute(
                        f'SELECT COUNT(DISTINCT "{col_name}") FROM "{table_name}"'
                    )
                    stats["distinct_count"] = cursor.fetchone()[0]

                    cursor.execute(
                        f'SELECT COUNT(*) FROM "{table_name}" WHERE "{col_name}" IS NULL'
                    )
                    stats["null_count"] = cursor.fetchone()[0]

                    if col_type in ("INTEGER", "REAL", "NUMERIC", "FLOAT", "DOUBLE"):
                        cursor.execute(
                            f'SELECT MIN("{col_name}"), MAX("{col_name}"), '
                            f'AVG("{col_name}") FROM "{table_name}"'
                        )
                        row = cursor.fetchone()
                        stats["min"] = row[0]
                        stats["max"] = row[1]
                        stats["avg"] = round(row[2], 2) if row[2] else None
                except Exception:
                    pass

                column_stats.append(stats)

            return {
                "table_name": table_name,
                "total_rows": total,
                "column_stats": column_stats,
            }

    @staticmethod
    def _cancel_query(conn):
        try:
            conn.interrupt()
        except Exception:
            pass


# ==================== MCP Server ====================

class DatabaseMCPServer:
    """数据库查询 MCP Server"""

    def __init__(self):
        self.db = DatabaseManager(Config.DB_PATH, Config.READONLY)
        self.security = SQLSecurityChecker()
        self.audit = AuditLogger(Config.AUDIT_LOG_PATH)

    def handle_request(self, request: dict) -> dict:
        method = request.get("method", "")
        req_id = request.get("id")

        handlers = {
            "initialize": self._handle_initialize,
            "tools/list": self._handle_list_tools,
            "tools/call": self._handle_call_tool,
        }

        handler = handlers.get(method)
        if handler:
            result = handler(request)
            return {"jsonrpc": "2.0", "id": req_id, "result": result}

        return {
            "jsonrpc": "2.0",
            "id": req_id,
            "error": {"code": -32601, "message": f"Unknown method: {method}"},
        }

    def _handle_initialize(self, request: dict) -> dict:
        return {
            "protocolVersion": "2024-11-05",
            "serverInfo": {
                "name": "database-query-server",
                "version": "1.0.0",
            },
            "capabilities": {"tools": {}},
        }

    def _handle_list_tools(self, request: dict) -> dict:
        return {
            "tools": [
                {
                    "name": "query_database",
                    "description": (
                        "执行只读 SQL 查询。只允许 SELECT 语句，"
                        "内置 SQL 注入防护和查询超时控制。"
                    ),
                    "inputSchema": {
                        "type": "object",
                        "properties": {
                            "sql": {
                                "type": "string",
                                "description": "要执行的 SELECT SQL 语句",
                            },
                            "max_rows": {
                                "type": "integer",
                                "description": "最大返回行数（默认1000）",
                                "default": 1000,
                            },
                        },
                        "required": ["sql"],
                    },
                },
                {
                    "name": "list_tables",
                    "description": "列出数据库中的所有表和视图，包含行数统计",
                    "inputSchema": {
                        "type": "object",
                        "properties": {},
                    },
                },
                {
                    "name": "describe_table",
                    "description": "查看指定表的结构信息，包含字段名、类型、索引等",
                    "inputSchema": {
                        "type": "object",
                        "properties": {
                            "table_name": {
                                "type": "string",
                                "description": "表名",
                            },
                        },
                        "required": ["table_name"],
                    },
                },
                {
                    "name": "get_statistics",
                    "description": (
                        "获取指定表的统计信息，包含行数、各列的"
                        "唯一值数量、空值数量、数值列的最大/最小/平均值"
                    ),
                    "inputSchema": {
                        "type": "object",
                        "properties": {
                            "table_name": {
                                "type": "string",
                                "description": "表名",
                            },
                        },
                        "required": ["table_name"],
                    },
                },
            ]
        }

    def _handle_call_tool(self, request: dict) -> dict:
        params = request.get("params", {})
        tool_name = params.get("name", "")
        arguments = params.get("arguments", {})

        tool_handlers = {
            "query_database": self._tool_query_database,
            "list_tables": self._tool_list_tables,
            "describe_table": self._tool_describe_table,
            "get_statistics": self._tool_get_statistics,
        }

        handler = tool_handlers.get(tool_name)
        if not handler:
            return self._error_response(f"未知工具: {tool_name}")

        try:
            result = handler(arguments)
            return {
                "content": [
                    {
                        "type": "text",
                        "text": json.dumps(result, ensure_ascii=False, default=str),
                    }
                ]
            }
        except TimeoutError as e:
            self.audit.log_query(
                str(arguments), False, 0, error="查询超时"
            )
            return self._error_response(f"查询超时（超过 {Config.QUERY_TIMEOUT} 秒）")
        except ValueError as e:
            return self._error_response(str(e))
        except Exception as e:
            return self._error_response("查询执行出错，请检查 SQL 语法")

    def _tool_query_database(self, args: dict) -> dict:
        sql = args.get("sql", "")
        max_rows = args.get("max_rows", Config.MAX_ROWS)

        is_safe, reason = self.security.check(sql)
        if not is_safe:
            self.audit.log_blocked(sql, reason)
            raise ValueError(f"安全检查未通过: {reason}")

        result = self.db.execute_query(sql, max_rows)

        self.audit.log_query(
            sql,
            result["success"],
            result["duration_ms"],
            result["row_count"],
        )

        return result

    def _tool_list_tables(self, args: dict) -> dict:
        tables = self.db.list_tables()
        self.audit.log_query("LIST TABLES", True, 0, len(tables))
        return {"tables": tables, "count": len(tables)}

    def _tool_describe_table(self, args: dict) -> dict:
        table_name = args.get("table_name", "")
        result = self.db.describe_table(table_name)
        self.audit.log_query(f"DESCRIBE {table_name}", True, 0)
        return result

    def _tool_get_statistics(self, args: dict) -> dict:
        table_name = args.get("table_name", "")
        result = self.db.get_statistics(table_name)
        self.audit.log_query(f"STATISTICS {table_name}", True, 0)
        return result

    @staticmethod
    def _error_response(message: str) -> dict:
        return {
            "content": [{"type": "text", "text": message}],
            "isError": True,
        }

    def run(self):
        """启动 MCP Server"""
        while True:
            try:
                line = sys.stdin.readline()
                if not line:
                    break

                request = json.loads(line.strip())
                response = self.handle_request(request)
                sys.stdout.write(json.dumps(response) + "\n")
                sys.stdout.flush()
            except json.JSONDecodeError:
                continue
            except KeyboardInterrupt:
                break


# ==================== 测试用例 ====================

def create_sample_database():
    """创建示例数据库用于测试"""
    conn = sqlite3.connect(Config.DB_PATH)
    cursor = conn.cursor()

    cursor.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            email TEXT UNIQUE NOT NULL,
            department TEXT,
            created_at TEXT DEFAULT (datetime('now')),
            is_active INTEGER DEFAULT 1
        )
    """)

    cursor.execute("""
        CREATE TABLE IF NOT EXISTS orders (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            user_id INTEGER NOT NULL,
            product TEXT NOT NULL,
            amount REAL NOT NULL,
            status TEXT DEFAULT 'pending',
            created_at TEXT DEFAULT (datetime('now')),
            FOREIGN KEY (user_id) REFERENCES users(id)
        )
    """)

    sample_users = [
        ("张三", "zhangsan@example.com", "工程部"),
        ("李四", "lisi@example.com", "产品部"),
        ("王五", "wangwu@example.com", "市场部"),
        ("赵六", "zhaoliu@example.com", "工程部"),
        ("陈七", "chenqi@example.com", "销售部"),
    ]

    cursor.executemany(
        "INSERT OR IGNORE INTO users (name, email, department) VALUES (?, ?, ?)",
        sample_users,
    )

    sample_orders = [
        (1, "云服务器 ECS", 2999.00, "completed"),
        (1, "对象存储 OSS", 599.00, "completed"),
        (2, "域名注册", 99.00, "pending"),
        (3, "CDN 加速", 1299.00, "completed"),
        (4, "数据库 RDS", 3999.00, "processing"),
        (5, "安全证书 SSL", 299.00, "completed"),
    ]

    cursor.executemany(
        "INSERT OR IGNORE INTO orders (user_id, product, amount, status) VALUES (?, ?, ?, ?)",
        sample_orders,
    )

    conn.commit()
    conn.close()


def run_security_tests():
    """运行安全检查测试"""
    checker = SQLSecurityChecker()

    test_cases = [
        ("SELECT * FROM users", True, "正常查询"),
        ("SELECT name FROM users WHERE id = 1", True, "带条件查询"),
        ("DROP TABLE users", False, "DROP 攻击"),
        ("DELETE FROM users", False, "DELETE 攻击"),
        ("SELECT * FROM users; DROP TABLE users", False, "多语句注入"),
        ("SELECT * FROM users UNION SELECT * FROM passwords", False, "UNION 注入"),
        ("SELECT * FROM users WHERE id = 1 OR 1=1", False, "OR 1=1 注入"),
        ("UPDATE users SET name = 'hacked'", False, "UPDATE 攻击"),
        ("SELECT * FROM users -- comment", False, "注释注入"),
        ("SELECT SLEEP(10)", False, "延时注入"),
        (
            "WITH cte AS (SELECT * FROM users) SELECT * FROM cte",
            True,
            "CTE 查询",
        ),
    ]

    print("=" * 60)
    print("SQL 安全检查测试")
    print("=" * 60)

    passed = 0
    total = len(test_cases)

    for sql, expected_safe, description in test_cases:
        is_safe, reason = checker.check(sql)
        status = "✅" if is_safe == expected_safe else "❌"
        if is_safe == expected_safe:
            passed += 1
        print(f"{status} {description}")
        print(f"   SQL: {sql}")
        print(f"   预期: {'安全' if expected_safe else '拦截'} | "
              f"实际: {'安全' if is_safe else '拦截'}")
        if not is_safe:
            print(f"   原因: {reason}")
        print()

    print(f"测试结果: {passed}/{total} 通过")


if __name__ == "__main__":
    if len(sys.argv) > 1 and sys.argv[1] == "--test":
        create_sample_database()
        run_security_tests()
    elif len(sys.argv) > 1 and sys.argv[1] == "--init-db":
        create_sample_database()
        print(f"示例数据库已创建: {Config.DB_PATH}")
    else:
        server = DatabaseMCPServer()
        server.run()
```
