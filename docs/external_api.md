---
icon: lucide/plug
---

# 外部 API 总览

JsMacros 调外部东西大概分两类：

| 类型 | 入口 | 例子 |
| --- | --- | --- |
| HTTP / WebSocket 服务 | `Request` | Webhook、机器人接口、本地后端、统计接口 |
| JVM / Mod Java API | `Java.type`、`Packages`、`Reflection` | Baritone、其他 Fabric/Forge/NeoForge Mod |

这两类都很有用，但也都比普通 `Player` / `World` API 更容易出事。写之前先想清楚：会不会泄露 token，会不会卡住事件线程，会不会因为版本更新直接炸。

## 调 HTTP API

最稳的基本模板：

```javascript
function getJson(url, headers = null) {
    const req = Request.create(url)
        .setConnectTimeout(3000)
        .setReadTimeout(5000)

    if (headers) {
        for (const key of headers.keySet()) {
            req.addHeader(key, headers.get(key))
        }
    }

    const res = req.get()
    return JSON.parse(res.text())
}

try {
    const data = getJson("https://example.com/api/status")
    Chat.log(data.message)
} catch (e) {
    Chat.log(`请求失败: ${e}`)
}
```

POST JSON：

```javascript
const headers = JavaUtils.createHashMap()
headers.put("Content-Type", "application/json")

const body = JSON.stringify({
    name: Player.getPlayer()?.getPlayerName(),
    time: Time.time(),
})

try {
    const res = Request.create("https://example.com/webhook")
        .addHeader("Content-Type", "application/json")
        .setConnectTimeout(3000)
        .setReadTimeout(5000)
        .post(body)

    Chat.log(res.text())
} catch (e) {
    Chat.log(`提交失败: ${e}`)
}
```

!!! tip "JSON 读取"
    `HTTPRequest$Response.json()` 在 2.1.0 类型里标为 deprecated。新脚本推荐 `JSON.parse(response.text())`。

## HTTP 调用守则

| 问题 | 建议 |
| --- | --- |
| 请求可能很慢 | 必须设置连接和读取超时 |
| 在 `Tick`、`RecvMessage` 等事件里请求 | 不要同步阻塞高频事件；先节流，或把耗时逻辑放到异步回调/主循环 |
| token、Cookie、Webhook URL | 放配置文件，别写进公开脚本 |
| 接口失败 | `try/catch`，给出本地提示，别让脚本静默死掉 |
| 高频请求 | 加冷却和缓存，别每 tick 请求一次 |

## 调其他 Mod 的 Java API

其他 Mod 如果把 API 类放在 classpath 里，可以用 `Java.type` 拿到。

```javascript
function tryJavaType(name) {
    try {
        return Java.type(name)
    } catch (e) {
        Chat.log(`找不到 Java 类: ${name}`)
        return null
    }
}

const SomeApi = tryJavaType("com.example.mod.api.SomeApi")
if (!SomeApi) return
```

先检查 Mod 是否加载：

```javascript
if (!Client.isModLoaded("baritone")) {
    Chat.log("没有加载 Baritone")
    return
}
```

!!! note "modId 不一定等于类名前缀"
    `Client.isModLoaded(modId)` 查的是 Mod ID。Java 类名看的是包名。两者经常相似，但不是同一个概念。

## 版本和映射

第三方 API 最容易踩这几个坑：

| 坑 | 说明 |
| --- | --- |
| Minecraft 原始类名变了 | `net.minecraft.class_2338` 这类 intermediary 名跟版本/映射有关 |
| Mod API 版本变了 | 方法名、返回值、包名可能变化 |
| API jar 和实现 jar 不一致 | 编译能过，运行可能找不到类或方法 |
| 直接访问非 API 包 | 维护者通常不保证兼容 |

写法上尽量：

- 优先调用公开 API 包，比如 `baritone.api.*`。
- 把外部 API 包在自己的小 class / 对象里，别散落在全脚本。
- 用 `try/catch` 包 `Java.type` 和关键调用。
- 在停止脚本时清理状态，比如取消寻路、松开强制按键、注销 HUD。