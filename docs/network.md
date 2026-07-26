---
icon: lucide/wifi
---

# 网络请求

`Request` 提供 HTTP 请求和 WebSocket。适合读取接口、提交简单数据、连接本地服务、对接推送服务。

如果你要系统性地接第三方 HTTP 接口、Webhook 或其他本地服务，可以继续看 [外部 API 总览](external_api.md)。想直接操作 Minecraft 协议层的数据包，请看 [数据包](packets.md)。

!!! warning "隐私提醒"
    不要把服务器 token、账号信息、聊天内容随手发到第三方接口。公开脚本前尤其要检查 URL 和 headers。

## Request 命名空间总览

以下方法均已对照 `JsMacros-2.1.0.d.ts` 核实：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `Request.create(url)` | `HTTPRequest` | 创建请求构建器，可加 header、设超时 |
| `Request.get(url)` | `HTTPRequest$Response` | 直接发 GET |
| `Request.get(url, headers)` | `HTTPRequest$Response` | 带请求头的 GET，`headers` 是 Java Map，可传 `null` |
| `Request.post(url, data)` | `HTTPRequest$Response` | 直接发 POST，`data` 为字符串 |
| `Request.post(url, data, headers)` | `HTTPRequest$Response` | 带请求头的 POST |
| `Request.createWS(url)` | `Websocket` | 创建 WebSocket 处理器 |
| `Request.createWS2(url)` | `Websocket` | 已弃用（1.2.7 起），用 `createWS` |

## 快速 GET：请求 JSON API 并解析

`Response.text()` 拿到文本后，用 JS 自带的 `JSON.parse` 解析即可。

```javascript
const resp = Request.get("https://api.github.com/repos/JsMacros/JsMacros")

if (resp.responseCode === 200) {
    const data = JSON.parse(resp.text())
    Chat.log(`仓库星标数: ${data.stargazers_count}`)
} else {
    Chat.log(`请求失败，状态码: ${resp.responseCode}`)
}
```

## 快速 POST

```javascript
const resp = Request.post("https://example.com/api", JSON.stringify({ hello: "world" }))
Chat.log(resp.text())
```

带 headers（`Request.get` / `Request.post` 的 headers 参数是 Java Map）：

```javascript
const headers = JavaUtils.createHashMap()
headers.put("Content-Type", "application/json")

const resp = Request.post("https://example.com/api", '{"ok":true}', headers)
Chat.log(resp.text())
```

## HTTPRequest：自定义 header 与鉴权

需要超时、多个 header、PUT 或自定义方法时，用 `Request.create`，链式调用很顺手：

```javascript
const resp = Request.create("https://example.com/api/profile")
    .addHeader("Accept", "application/json")
    .addHeader("Authorization", "Bearer 你的token")
    .setConnectTimeout(3000)
    .setReadTimeout(5000)
    .get()

const data = JSON.parse(resp.text())
Chat.log(data.name)
```

`HTTPRequest` 的完整成员：

| 成员 | 作用 |
| --- | --- |
| `addHeader(key, value)` | 添加请求头，返回自身可链式调用 |
| `setConnectTimeout(timeout)` | 连接超时（毫秒），返回自身 |
| `setReadTimeout(timeout)` | 读取超时（毫秒），返回自身 |
| `get()` | 发送 GET |
| `post(data)` | 发送 POST，`data` 可为字符串或字节数组 |
| `put(data)` | 发送 PUT，`data` 可为字符串或字节数组 |
| `send(method)` | 发送自定义方法（无请求体），如 `"DELETE"` |
| `send(method, data)` | 自定义方法 + 请求体（字符串或字节数组） |
| `headers` 字段 | 已设置的请求头 Map |
| `connectTimeout` / `readTimeout` 字段 | 当前超时设置 |

!!! warning "别把 token 写死在脚本里"
    脚本文件经常被整个文件夹打包分享、直播时露出、提交到 Git 仓库。写死在脚本里的
    API key、推送 token 一旦泄露就可能被人冒用。建议把 token 放到脚本目录外的单独
    配置文件里读取，分享脚本前务必检查。

    顺带一提：很多人用这套 HTTP 接口对接"消息推送服务"（如 PushPlus、Server酱 等），
    原理就是把文本 POST 到服务商的 URL（带上你的 token），服务商再推送到你的微信/手机。
    这类 token 同样属于敏感信息。

## Response

`HTTPRequest$Response` 的完整成员：

| 成员 | 作用 |
| --- | --- |
| `text()` | 以文本读取响应体 |
| `byteArray()` | 以字节数组读取响应体（下载二进制文件用） |
| `json()` | 已弃用，官方注释明确说"别用它"，请在脚本语言里自己解析 `text()` |
| `responseCode` 字段 | HTTP 状态码，如 `200`、`404` |
| `headers` 字段 | 响应头，`Map<String, List<String>>`，可能为 `null` |

!!! tip "JSON 推荐写法"
    `response.json()` 在 2.1.0 类型里标为 deprecated。推荐 `JSON.parse(response.text())`。

!!! note "响应体建议只取一次"
    从类型定义看，响应体来自输入流（`InputStream`），流式数据通常读一次就被消费掉。
    建议对同一个响应只调用一次 `text()` 或 `byteArray()`，结果存变量复用
    （重复调用的具体行为以实测为准）。

## 异步注意：网络请求会阻塞脚本线程

`Request.get` / `post` / `send` 都是**同步阻塞**的：脚本会停在那一行直到服务器返回或超时。

- 普通脚本、`joined = false` 的事件回调各自跑在独立线程上，阻塞的只是脚本自己，游戏不卡。
- **`joined = true` 的回调会阻塞游戏事件线程**，在里面发请求会导致游戏卡顿甚至触发看门狗杀脚本。

```javascript
// 错误示范：joined 回调里发请求，游戏会卡住等网络
JsMacros.on("SendMessage", true, JavaWrapper.methodToJava((e, context) => {
    Request.post("https://example.com/log", e.message)  // 不要这样做！
}))

// 正确做法一：不需要取消/修改事件时，用普通（非 joined）监听
JsMacros.on("SendMessage", JavaWrapper.methodToJava((e) => {
    Request.post("https://example.com/log", e.message)  // 独立线程，随便慢
}))

// 正确做法二：必须 joined 时，先处理完事件、释放锁，再发请求
JsMacros.on("SendMessage", true, JavaWrapper.methodToJava((e, context) => {
    e.cancel()
    context.releaseLock()   // 先放行游戏线程
    Request.post("https://example.com/log", "已拦截一条消息")
}))
```

给不熟悉 `joined` 参数的读者：详见[事件系统](events.md)。

## WebSocket

`Request.createWS(url)` 返回 `Websocket` 对象。它的回调（`onConnect`、`onTextMessage` 等）
是**需要赋值的字段**，不是方法——把 `JavaWrapper.methodToJava(...)` 包装的函数赋给它们，
并且要**在调用 `connect()` 之前**赋好。

### 完整生命周期示例

```javascript
const ws = Request.createWS("wss://example.com/socket")

// 连接成功。参数：底层 WebSocket 对象、响应头 Map
ws.onConnect = JavaWrapper.methodToJava((socket, headers) => {
    Chat.log("WebSocket 已连接")
    ws.sendText("hello")
})

// 收到文本消息。参数：底层 WebSocket 对象、消息字符串
ws.onTextMessage = JavaWrapper.methodToJava((socket, msg) => {
    Chat.log(`收到: ${msg}`)
})

// 连接断开。参数：底层 WebSocket 对象、Disconnected 信息
ws.onDisconnect = JavaWrapper.methodToJava((socket, info) => {
    Chat.log(`连接断开，是否服务器主动断开: ${info.isServer}`)
})

// 出错
ws.onError = JavaWrapper.methodToJava((socket, err) => {
    Chat.log(`WebSocket 出错: ${err}`)
})

ws.connect()          // 开始连接，返回自身
ws.sendText("再发一条")  // 发送文本，返回自身

// 用完记得关
// ws.close()         // 或 ws.close(1000) 带状态码
```

### Websocket 成员一览

回调字段（都可为 `null`，赋 `MethodWrapper`）：

| 字段 | 回调参数 | 触发时机 |
| --- | --- | --- |
| `onConnect` | `(socket, headers)` | 连接建立，`headers` 为 `Map<String, List<String>>` |
| `onTextMessage` | `(socket, text)` | 收到文本消息 |
| `onDisconnect` | `(socket, disconnected)` | 连接断开 |
| `onError` | `(socket, exception)` | 发生异常 |
| `onFrame` | `(socket, frame)` | 收到任意底层帧（高级用法） |

方法：

| 方法 | 作用 |
| --- | --- |
| `connect()` | 连接，返回自身；失败抛 `WebSocketException` |
| `sendText(text)` | 发送文本，返回自身 |
| `close()` / `close(closeCode)` | 关闭连接，返回自身 |
| `getWs()` | 取得底层 `com.neovisionaries.ws.client.WebSocket` 对象（高级用法） |

`onDisconnect` 收到的 `Websocket$Disconnected` 对象：

| 字段 | 含义 |
| --- | --- |
| `serverFrame` | 服务器侧的关闭帧 |
| `clientFrame` | 客户端侧的关闭帧 |
| `isServer` | 是否由服务器发起断开 |

### 断线重连骨架

重连时建议直接新建一个 `Websocket` 对象并重新赋回调（复用同一对象反复 `connect()` 的行为以实测为准）：

```javascript
let ws = null
let stopped = false

function connect() {
    ws = Request.createWS("wss://example.com/socket")

    ws.onTextMessage = JavaWrapper.methodToJava((socket, msg) => {
        Chat.log(`收到: ${msg}`)
    })

    ws.onDisconnect = JavaWrapper.methodToJava(() => {
        if (stopped) return
        Chat.log("连接断开，5 秒后重连...")
        Time.sleep(5000)
        connect()
    })

    ws.onError = JavaWrapper.methodToJava((socket, err) => {
        Chat.log(`出错: ${err}`)
    })

    ws.connect()
}

connect()
```

!!! tip "长连接建议做成服务"
    WebSocket 需要脚本一直活着才能收到回调。一次性脚本跑完主体就可能被回收，
    建议把长连接脚本注册成[服务](services.md)，随游戏启动、可手动开关，
    停止时置 `stopped = true` 并调用 `ws.close()`。
