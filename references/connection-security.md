# 服务连接与凭据安全

在发送 Bearer Token 或任何客户数据之前，先验证 `LETU_KB_BASE_URL`。

## 允许的服务地址

1. 使用标准 URL 解析器解析地址；解析失败立即停止。
2. URL 不得包含用户名或密码（URL `credentials`），不得包含 `fragment`/片段。
3. 生产服务必须使用 HTTPS。
4. HTTP 只允许本机测试，Host 必须精确为 `localhost`、`127.0.0.1` 或 `[::1]`；相似域名、子域名和其他私网地址都不属于 loopback/回环例外。
5. 首次使用时向用户显示不含路径、查询和凭据的 `origin`，由用户确认这是其组织提供的 leto AI 服务。确认后，本次任务固定使用该 Origin。

不要把 Token 放进 URL、查询参数、文件名、命令行回显或异常消息。

## 请求和重定向

- API Path 必须相对于已经确认的 Origin 解析。
- 携带 `Authorization` 的请求使用禁止自动重定向的模式。收到任何 3xx 都停止，并让用户核对服务地址。
- 绝不向跨源地址发送或转发 `Authorization`。跨源包括协议、Host 或 Port 任一变化。
- Agent API 当前返回的上传 URL 必须是以 `/` 开头的同源相对路径。若返回绝对 URL、`//host/path` 或其他 Origin，停止上传；不要猜测 OSS 地址。
- 对已经校验为同源相对路径的上传请求，发送通用 `Authorization: Bearer $LETU_KB_API_TOKEN`，并合并服务端明确返回的上传 Headers。不要添加 Cookie、代理凭据或其他环境变量内容。若未来协议返回跨源上传 URL，必须停止并重新读取 Bootstrap/本安全契约，绝不能把 Authorization 复制到该地址。

## 安全失败

- URL 校验失败、TLS 失败、证书异常、重定向或 Origin 变化都应失败关闭。
- 不允许为了“先试一下”关闭证书校验。
- 401 表示 Token 缺失、无效或已撤销；403 表示 Scope 不足。不要改用其他接口绕过。
- 日志仅保留 HTTP 状态、稳定错误码和服务端 Request ID，不保存 Header、Token、完整 URL、问题正文或客户文件内容。
