---
icon: lucide/monitor
---

# 客户端

`Client` 负责 Minecraft 客户端级别的能力：等待 tick、主线程执行、连接服务器、剪贴板、Mod 列表、注册表和网络包。

## Tick 和主线程

```javascript
Client.waitTick()
Client.waitTick(20)
```

`waitTick()` 等 1 tick，`waitTick(i)` 等 i tick。长循环里优先用它，而不是疯狂 `while(true)`。

```javascript
Client.runOnMainThread(JavaWrapper.methodToJava(() => {
    Chat.log("这段在主线程执行")
}))
```

`runOnMainThread` 有 `watchdogMaxTime` 和 `await` 重载。只把必须在主线程做的 Minecraft 操作放进去。

!!! warning "别在 joined 事件里乱等"
    joined 事件已经可能阻塞主线程。此时再 `Client.waitTick()` 或等待主线程任务，容易把自己锁住。

## 客户端状态

```javascript
Chat.log(Client.mcVersion())
Chat.log(Client.getFPS())
Chat.log(Client.getModLoader())
Chat.log(Client.isDevEnv())
```

| 方法 | 作用 |
| --- | --- |
| `mcVersion()` | Minecraft 版本 |
| `getFPS()` | 当前 FPS 文本 |
| `getGameOptions()` | 游戏设置 helper |
| `getMinecraft()` | 原始 Minecraft 客户端对象 |
| `getRegistryManager()` | 注册表 helper |

## 连接和退出

```javascript
Client.connect("example.com")
Client.connect("127.0.0.1", 25565)
Client.disconnect()
```

服务器 ping：

```javascript
const info = Client.ping("example.com")
Chat.log(`${info.getLabel().getString()} ${info.getPing()}`)
```

异步 ping：

```javascript
Client.pingAsync("example.com", JavaWrapper.methodToJava((info, err) => {
    if (err) Chat.log(err)
    else Chat.log(info.getPing())
}))
```

退出游戏：

```javascript
Client.exitGamePeacefully()
// Client.exitGameForcefully()
```

## Mod 和注册表

```javascript
if (Client.isModLoaded("jsmacros")) {
    Chat.log("JsMacros 已加载")
}

const mods = Client.getLoadedMods()
Chat.log(`已加载 Mod 数量: ${mods.size()}`)
```

注册表速查：

```javascript
const items = Client.getRegisteredItems()
const blocks = Client.getRegisteredBlocks()
Chat.log(`${items.size()} items, ${blocks.size()} blocks`)
```

## 剪贴板

```javascript
Client.setClipboard("复制到剪贴板")
Chat.log(Client.getClipboard())
```

## 网络包

`sendPacket(packet)` 和 `receivePacket(packet)` 接收原始 Minecraft packet 对象。

!!! warning "高级危险区"
    网络包 API 很容易触发反作弊或造成客户端状态不同步。除非你清楚包结构和服务端规则，否则不要在多人服务器里使用。

