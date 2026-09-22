# MCP 客户端配置示例

> `<网关根地址>` = 机主给的网关 URL 去掉 `/mcp` 的部分，形如 `http://101.69.x.x:28471`（http 模式）或 `https://域名:端口`（TLS 模式）。

把 `<域名>` 和 `<密钥>` 换成机主发给你的实际值。三种写法按优先级尝试，成功一种即可。

## 1. Streamable HTTP（首选，大多数现代客户端支持）

```json
{
  "mcpServers": {
    "mac-gateway": {
      "type": "http",
      "url": "<网关根地址>/mcp",
      "headers": { "Authorization": "Bearer <密钥>" }
    }
  }
}
```

## 2. SSE（客户端只认旧传输时）

```json
{
  "mcpServers": {
    "mac-gateway": {
      "type": "sse",
      "url": "<网关根地址>/sse",
      "headers": { "Authorization": "Bearer <密钥>" }
    }
  }
}
```

## 3. 查询参数兜底（客户端不支持自定义请求头时）

```json
{
  "mcpServers": {
    "mac-gateway": {
      "type": "http",
      "url": "<网关根地址>/mcp?key=<密钥>"
    }
  }
}
```

注意：`?key=` 会把密钥带在 URL 上，可能进对端访问日志，仅在无法用请求头时使用，
并提醒机主可以通过 `revoke-key` 轮换。

## 常见客户端字段名差异

- 有的客户端用 `"transport": "http"` 而不是 `"type"`；有的 sse 条目不需要 `headers` 字段名而用 `"headers"` 同义的配置块
- 拿不准就看客户端文档里 MCP server 的最小可运行示例，保持它的字段结构，只替换 url 与鉴权
- 配置完成后先调 `now_playing` 验证；401 说明密钥不对或已被吊销，connection 错误说明域名/端口/转发没通
