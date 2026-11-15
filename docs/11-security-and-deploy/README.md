# 11 - 安全与部署

> 已按本地 nanobot `0.4.1`、提交 `27f1a836b5c04a30763dd91960059f81c05bf505` 核对。方法使用本地新版完整源码并配中文 # 注释解读，配置示例单独标注。详见 [版本核对记录](../LOCAL_SOURCE_BASELINE.md)。

部署命令与安全说明按本地实现核对；不再把正则黑名单、cwd或Prompt误称完备沙箱。未实际部署的内容仍为操作说明。

## 目录

- [11.1 安全边界而非提示词承诺](#section-11-1)
- [11.2 真实路径处理](#section-11-2)
- [11.3 Shell、沙箱与Windows](#section-11-3)
- [11.4 网络与MCP](#section-11-4)
- [11.5 密钥与访问](#section-11-5)
- [11.6 当前Docker入口](#section-11-6)
- [11.7 备份、日志与恢复](#section-11-7)
- [11.8 验证与面试](#section-11-8)

<a id="section-11-1"></a>

## 11.1 安全边界而非提示词承诺

模型提出调用，后端负责授权。工作区路径规则、Schema、进程沙箱、网络检查与平台访问是不同防线；SSRF不是Agent专有漏洞，但工具让不可信内容影响访问意图的路径值得关注。


<a id="section-11-2"></a>

## 11.2 真实路径处理

<a id="method-is-path-within"></a>

源码：[nanobot/security/workspace_policy.py:43](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/security/workspace_policy.py#L43)，`is_path_within` 完整方法。

```python
# 解析真实路径层级关系，判断目标是否位于允许根目录内。
def is_path_within(path: str | Path, root: str | Path) -> bool:
    """Return True when *path* resolves to *root* or a descendant of *root*."""
    try:
        resolved_path = Path(path).expanduser().resolve(strict=False)
        resolved_root = Path(root).expanduser().resolve(strict=False)
        # 按解析后的目录层级判断包含关系，不使用字符串前缀。
        resolved_path.relative_to(resolved_root)
        return True
    # 处理 (OSError, RuntimeError, TypeError, ValueError)，按本方法约定记录、转换或继续传播错误。
    except (OSError, RuntimeError, TypeError, ValueError):
        return False
```

当前通过归一化/resolve与relative_to判断关系，具体工具结合WorkspaceScope与授权。旧的字符串startswith示意既不是当前实现，也不能正确覆盖同前缀目录、符号链接等情况。跨根读取技能/指定历史可有明确只读能力，不等于整个Agent工作区都可写。


<a id="section-11-3"></a>

## 11.3 Shell、沙箱与Windows

ExecTool有命令过滤、运行时间、输出、进程会话、取消与sandbox配置。allow_patterns/deny_patterns不是对任意命令语义的完备证明；设cwd不能限制命令访问外部文件。真正隔离还取决于所选OS后端与宿主权限。

Windows默认优先pwsh/PowerShell。不要拿Linux的bwrap设置宣称Windows自动有同等隔离。缺少/失败的已配置沙箱启动器不应回退成无沙箱执行。

[nanobot/agent/tools/shell.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/shell.py) · [nanobot/agent/tools/sandbox.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/agent/tools/sandbox.py)


<a id="section-11-4"></a>

## 11.4 网络与MCP

security/network与web/MCP路径核对URL、解析目标、代理/重定向及白名单等。不能只检查URL文本中的localhost；也不应遇拒绝后通过curl、别名IP等绕过。远端MCP是额外信任边界，其只读声明和OAuth不自动证明内容可信。

[nanobot/security/network.py](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/nanobot/security/network.py) · [docs/guides/configure-mcp-tools.md](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docs/guides/configure-mcp-tools.md)


<a id="section-11-5"></a>

## 11.5 密钥与访问

config可引用${VAR}，但设置环境变量不代表配置自动读取任意同名Key；服务环境需真正继承引用。OAuth与直接API Key也有不同持久化路径。不要把认证链接、token、真实config或数据库提交到Git。

WebUI管理一个共享实例，不承诺每个登录者有私有会话与独立管理员角色。渠道允许名单/配对与多租户隔离不同；多用户不应仅靠session_key混接共享权限。


<a id="section-11-6"></a>

## 11.6 当前Docker入口

从**上游源码根目录**使用当前Dockerfile/Compose，不复制原教程虚构的服务与挂载路径。以下是有副作用的部署命令，读者决定部署后再运行：

```bash
docker compose build
docker compose up -d nanobot-gateway
docker compose logs --tail=100 nanobot-gateway
```

官方Compose挂载~/.nanobot到/home/nanobot/.nanobot，health默认18790，WebUI/WebSocket默认8765。非root运行、bootstrap能力与no-new-privileges各有作用；使用额外bwrap覆盖文件仍需宿主允许命名空间。镜像不自动提供多租户安全或严格离线。

[docker-compose.yml](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docker-compose.yml) · [Dockerfile](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/Dockerfile) · [docker-compose.bwrap.yml](https://github.com/HKUDS/nanobot/blob/27f1a836b5c04a30763dd91960059f81c05bf505/docker-compose.bwrap.yml)


<a id="section-11-7"></a>

## 11.7 备份、日志与恢复

备份要区分SQLite会话、配置、Agent资料与项目文件。运行中的SQLite使用其支持的一致性方式，不仅复制单一主文件而忽略WAL；恢复先验证实例、命名空间与凭据，不能直接重放未知副作用。

日志短预览用于诊断，不代替历史备份。工具Schema、会话、模型来源、执行/交付状态应分层定位；没有测量不宣称支持某个并发规模。


<a id="section-11-8"></a>

## 11.8 验证与面试

读test_workspace_policy、test_security_network、test_sandbox和gateway测试。只对覆盖路径下结论，平台条件跳过不算通过。实际练习仅用自建文件，不为了观察拒绝读取凭据或真实内网。

回答为什么路径规则不等于OS沙箱，为什么取消不自动回滚，为什么页面断连后不直接重做工具。

## 下一章

[继续第 12 章](../12-nanobot-real-projects/README.md)
