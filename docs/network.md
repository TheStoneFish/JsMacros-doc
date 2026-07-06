---
icon: lucide/wifi
---

# 网络请求

`Request` 提供 HTTP 请求和 WebSocket。适合读取接口、提交简单数据、连接本地服务。

如果你要系统性地接第三方 HTTP 接口、Webhook 或其他本地服务，可以继续看 [外部 API 总览](external_api.md)。

!!! warning "隐私提醒"
    不要把服务器 token、账号信息、聊天内容随手发到第三方接口。公开脚本前尤其要检查 URL 和 headers。

## 快速 GET / POST

```javascript
const res = Request.get("https://example.com/api")
Chat.log(res.text())
```

```javascript
const res = Request.post("https://example.com/api", JSON.stringify({ hello: "world" }))
Chat.log(res.text())
```

带 headers：

```javascript
const headers = JavaUtils.createHashMap()
headers.put("Content-Type", "application/json")

const res = Request.post("https://example.com/api", '{"ok":true}', headers)
Chat.log(res.text())
```

## HTTPRequest

需要超时、headers 或自定义方法时，用 `Request.create`。

```javascript
const req = Request.create("https://example.com/api")
    .addHeader("Accept", "application/json")
    .setConnectTimeout(3000)
    .setReadTimeout(5000)

const res = req.get()
const data = JSON.parse(res.text())
Chat.log(data.name)
```

| 方法 | 作用 |
| --- | --- |
| `addHeader(key, value)` | 添加请求头 |
| `setConnectTimeout(timeout)` | 连接超时，毫秒 |
| `setReadTimeout(timeout)` | 读取超时，毫秒 |
| `get()` / `post(data)` / `put(data)` | 常用请求 |
| `send(method)` / `send(method, data)` | 自定义方法 |

## Response

| 方法 | 作用 |
| --- | --- |
| `text()` | 文本响应 |
| `byteArray()` | 字节数组 |
| `json()` | 已弃用 |

!!! tip "JSON 推荐写法"
    `response.json()` 在 2.1.0 类型里标为 deprecated。推荐 `JSON.parse(response.text())`。

## WebSocket

```javascript
const ws = Request.createWS("wss://example.com/socket")
ws.connect()
ws.sendText("hello")
```

常用方法：

| 方法 | 作用 |
| --- | --- |
| `connect()` | 连接 |
| `sendText(text)` | 发送文本 |
| `close()` | 关闭 |
| `close(closeCode)` | 带状态码关闭 |
| `getWs()` | 取得底层 WebSocket 对象 |

复杂 WebSocket 回调通常需要直接操作底层 Java WebSocket，对新手不太友好，建议先把 HTTP 用熟。
