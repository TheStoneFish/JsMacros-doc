---
icon: lucide/radio
---

# 事件系统

事件系统是 JsMacros 的核心：聊天、按键、Tick、打开容器、收发数据包……游戏里的各种时机都会"广播"一个事件，脚本可以监听这些事件并自动运行。本页讲解事件机制本身——注册、注销、joined、等待事件、自定义事件和监听器清理。全部事件的字段与方法请看 [全部事件参考](events_reference.md)。

## 两种使用事件的方式

**方式一：在 JsMacros 界面里把脚本绑定到事件。** 这种脚本运行时自带两个全局变量：

| 全局变量 | 类型 | 说明 |
| --- | --- | --- |
| `event` | `Events.BaseEvent` | 触发本次运行的事件对象 |
| `context` | `EventContainer` | 本次运行的事件容器（锁）|
| `file` | `java.io.File` | 当前脚本文件 |

```javascript
// 绑定到 RecvMessage 事件的脚本
JsMacros.assertEvent(event, "RecvMessage") // 断言事件类型（TS 中还能获得补全）
if (event.text) {
    Chat.log(`收到: ${event.text.getString()}`)
}
```

**方式二：在任意脚本里用 `JsMacros.on` 动态注册监听器。** 这是服务（Service）脚本最常用的方式，也是本页的重点。

!!! note "回调必须用 JavaWrapper 包装"
    传给 `JsMacros.on/once/waitForEvent` 的回调，必须用 `JavaWrapper.methodToJava(...)` 包装成 Java 可调用的 `MethodWrapper`，直接传 JS 函数会报错。

## 回调的两个参数

监听回调总是收到两个参数 `(event, context)`：

- `event`——事件对象，字段因事件而异，通用方法只有 `getEventName()`；
- `context`——`EventContainer`，代表这次回调占用的"事件锁"，joined 监听时用它 `releaseLock()` 提前放行游戏线程。

```javascript
JsMacros.on("Tick", JavaWrapper.methodToJava((e, ctx) => {
    // e.getEventName() === "Tick"
}))
```

## 注册监听器

### JsMacros.on

`on` 返回一个 `IEventListener`，留着它才能之后 `off` 掉。全部重载（对照 d.ts）：

| 重载 | 说明 |
| --- | --- |
| `on(event, callback)` | 最简单形式，回调异步运行（joined = false）|
| `on(event, joined, callback)` | 指定是否加入（阻塞）事件线程 |
| `on(event, filter, callback)` | 附带 Java 端 `EventFilter` 过滤器 |
| `on(event, filter, joined, callback)` | 过滤器 + joined |

```javascript
const listener = JsMacros.on("Key", JavaWrapper.methodToJava((e, ctx) => {
    if (e.key === "key.keyboard.g" && e.action === 1) {
        Chat.log("按下了 G")
    }
}))
```

!!! tip "on 的 filter 是 EventFilter，不是 JS 函数"
    `on` 的过滤器参数必须是 `JsMacros.eventFilters()` 创建的 Java 端过滤器（见下文），在事件线程上以 Java 速度先筛一遍，脚本回调只在通过时才执行。想用 JS 函数过滤，请用 `waitForEvent`。

### JsMacros.once

只触发一次后自动注销：

| 重载 | 说明 |
| --- | --- |
| `once(event, callback)` | 一次性监听 |
| `once(event, joined, callback)` | 一次性 + joined |

```javascript
JsMacros.once("JoinServer", JavaWrapper.methodToJava((e) => {
    Chat.log(`进入服务器: ${e.address}`)
}))
```

### JsMacros.off 与 listeners

| 方法 | 说明 |
| --- | --- |
| `off(listener)` | 注销监听器，返回是否成功 |
| `off(event, listener)` | 从指定事件上注销，效率更高 |
| `listeners(event)` | 返回该事件上所有脚本添加的监听器列表 |

```javascript
const listener = JsMacros.on("Tick", JavaWrapper.methodToJava(() => {}))
JsMacros.off("Tick", listener)

Chat.log(`Tick 监听器数量: ${JsMacros.listeners("Tick").size()}`)
```

监听器对象（`IEventListener`）自身也有几个方法：`joined()` 查询是否为 joined 监听、`off()` 自我注销（适合"满足条件后不再监听"的写法）。

```javascript
let count = 0
const l = JsMacros.on("Tick", JavaWrapper.methodToJava(() => {
    if (++count >= 100) {
        Chat.log("100 tick 到了，停止监听")
        l.off() // 自我注销
    }
}))
```

## joined 参数深入

`joined` 决定回调在哪条线程上运行：

| joined | 行为 | 适合场景 |
| --- | --- | --- |
| `false`（默认） | 回调在独立线程异步运行，不阻塞游戏 | Tick 统计、HUD 更新、日志、普通提示 |
| `true` | 回调"加入"触发事件的线程，事件会等回调放行后才继续 | 取消事件、修改事件字段 |

**哪些事件 joined 才有意义？** 两类：

1. **可取消事件**（实现了 `Cancellable`，共 15 个）：`ClickSlot`、`DropSlot`、`Key`、`MouseScroll`、`NameChange`、`OpenContainer`、`Particle`、`RecvMessage`、`RecvPacket`、`SendMessage`、`SendPacket`、`ServerSound`、`SignEdit`、`Sound`、`Title`。只有 joined 监听里调用 `event.cancel()` 才来得及拦截。
2. **有可写字段的事件**：如 `SendMessage.message`、`RecvMessage.text`、`Title.message`、`NameChange.newName`、`SignEdit.signText`、`RecvPacket.packet` / `SendPacket.packet` 等，joined 时修改才会生效。

其余事件（`Tick`、`EntityLoad`、`HealthChange`……）用 joined 只会白白卡住游戏线程，保持 `false` 即可。

!!! warning "看门狗：joined 回调不要拖"
    joined 回调阻塞的是游戏线程。JsMacros 有看门狗机制：joined 回调占用锁过久（约 500 毫秒）会被强制释放并在日志中警告，游戏继续运行，你的修改可能已经错过时机。正确姿势是**先取消/修改事件，马上 `context.releaseLock()`，再做耗时工作**。

```javascript
JsMacros.on("SendMessage", true, JavaWrapper.methodToJava((e, context) => {
    if (e.message && e.message.startsWith(".local")) {
        e.cancel()             // 1. 先拦截
        context.releaseLock()  // 2. 立刻放行游戏线程
        // 3. 之后再慢慢做耗时的事
        Chat.log(`本地命令: ${e.message}`)
    }
}))
```

### EventContainer 方法表

`context` 参数（以及 `waitForEvent` 返回值里的 `context`）是 `EventContainer`，方法如下：

| 方法 | 说明 |
| --- | --- |
| `releaseLock()` | 提前释放事件锁，joined 监听的标配 |
| `isLocked()` | 是否仍持有锁 |
| `getCtx()` | 获取脚本上下文（`BaseScriptContext`，见 [脚本上下文](script_context.md)）|
| `getLockThread()` | 获取持锁线程 |
| `setLockThread(thread)` | 设置持锁线程（一般用不到）|
| `awaitLock(then)` | 等待锁释放后执行 `then`（`MethodWrapper`）。**小心死锁**，脚本里少用 |

## 等待事件：waitForEvent

`JsMacros.waitForEvent` 会**阻塞当前脚本线程**，直到事件发生，非常适合写"顺序流程"的脚本。全部重载：

| 重载 | 说明 |
| --- | --- |
| `waitForEvent(event)` | 等待下一次事件 |
| `waitForEvent(event, join)` | 以 joined 方式等待（返回后仍持有锁）|
| `waitForEvent(event, filter)` | 带过滤器，`filter` 是返回布尔值的 `MethodWrapper` |
| `waitForEvent(event, join, filter)` | joined + 过滤器 |
| `waitForEvent(event, filter, runBeforeWaiting)` | `runBeforeWaiting` 在开始等待前执行，防止竞态 |
| `waitForEvent(event, join, filter, runBeforeWaiting)` | 全参数版 |

返回值是 `EventAndContext` 结构：

| 字段 | 说明 |
| --- | --- |
| `event` | 等到的事件对象 |
| `context` | 对应的 `EventContainer`；如果是 joined 等待，用 `context.releaseLock()` 提前放行 |

```javascript
Chat.log("请按任意键...")
const result = JsMacros.waitForEvent("Key")
Chat.log(`按键: ${result.event.key}`)
```

带过滤器（注意：`waitForEvent` 的过滤器是 JS 回调，返回 `true` 才算等到）：

```javascript
// 等待一条包含"完成"的聊天消息
const result = JsMacros.waitForEvent(
    "RecvMessage",
    JavaWrapper.methodToJava((e) => e.text != null && e.text.getString().includes("完成"))
)
Chat.log(`等到了: ${result.event.text.getString()}`)
```

joined 等待并提前放行：

```javascript
const result = JsMacros.waitForEvent("Title", true)
Chat.log(`标题类型: ${result.event.type}`)
result.context.releaseLock() // joined 等待必须记得放行
```

!!! note "waitForEvent 与已持有的锁"
    如果当前线程已经绑定在某个事件上（比如在 joined 回调里调用），`waitForEvent` 会先释放当前锁再开始等待。另外它会抛出 `InterruptedException`——脚本被停止时等待会被打断，属正常现象。

## 事件过滤器（EventFilter）

`JsMacros.eventFilters()`（2.1.0 新增）返回过滤器工厂，创建的过滤器在 Java 端执行，用于 `on` 的 `filter` 参数，比在回调里手动 `if` 高效得多：

| 工厂方法 | 作用 |
| --- | --- |
| `modulus(n)` | 每 n 次事件放行 1 次 |
| `limited(n)` | 只放行前 n 次 |
| `invert(filter)` | 反转过滤器结果 |
| `composed(initial)` | 组合多个过滤器（与/或逻辑）|
| `compile(code)` / `compile(event, code)` / `compile(event, ...conditions)` | 把 Java 代码编译成过滤器 |

```javascript
// 每 20 tick 执行一次（约每秒）
JsMacros.on("Tick", JsMacros.eventFilters().modulus(20), JavaWrapper.methodToJava(() => {
    // 每秒一次的轮询
}))

// 只对特定按键触发
const filter = JsMacros.eventFilters().compile("Key", 'event.action == 1 && eq(event.key, "key.keyboard.g")')
JsMacros.on("Key", filter, JavaWrapper.methodToJava(() => Chat.log("G!")))
```

详细用法见 [事件过滤器](event_filters.md)。

## 自定义事件（Custom）

`JsMacros.createCustomEvent(eventName)` 创建一个 `Events.Custom` 对象，可以携带数据并触发，是**脚本间通信**的标准方案。

!!! tip "起名别撞车"
    不要使用已有事件名（如 `"Tick"`），否则其他脚本会被你搞糊涂。建议加前缀，如 `"myMod:refresh"`。

### 方法全表（以 d.ts 为准）

| 分类 | 成员 | 说明 |
| --- | --- | --- |
| 字段 | `eventName` | 事件名 |
| 字段 | `joinable` / `cancelable` | 是否可 join / 可取消（可赋值的标志位）|
| 查询 | `joinable()` / `cancellable()` | 以方法形式查询上述标志 |
| 触发 | `trigger()` | 同步触发。不要在自己的监听器里再次触发同一事件，会无限循环 |
| 触发 | `triggerAsync(callback)` | 触发并在所有监听器执行完后调用 `callback`（无参 Runnable）|
| 写入 | `putInt(name, i)` / `putString(name, str)` / `putDouble(name, d)` / `putBoolean(name, b)` / `putObject(name, o)` | 按类型存值 |
| 读取 | `getInt(name)` / `getString(name)` / `getDouble(name)` / `getBoolean(name)` / `getObject(name)` | 按类型取值，不存在返回 `null` |
| 读取 | `getType(name)` | 返回 `'Int'`/`'String'`/`'Double'`/`'Boolean'`/`'Object'` 或 `null` |
| 读取 | `getUnderlyingMap()` | 返回底层 `JavaMap<string, any>` |
| 注册 | `registerEvent()` | 把事件名注册进 GUI 事件列表，便于绑定脚本 |

### 脚本间通信示例

脚本 A（发送方）：

```javascript
const ev = JsMacros.createCustomEvent("myScripts:report")
ev.putString("from", "scriptA")
ev.putInt("hp", Math.round(Player.getPlayer().getHealth()))
ev.putBoolean("urgent", true)
ev.trigger()
```

脚本 B（接收方，比如一个常驻服务）：

```javascript
JsMacros.on("myScripts:report", JavaWrapper.methodToJava((e) => {
    Chat.log(`来自 ${e.getString("from")} 的报告: HP=${e.getInt("hp")}`)
    if (e.getBoolean("urgent")) {
        Chat.log("§c紧急!")
    }
}))
```

想在 GUI 里直接把脚本绑定到自定义事件，先执行一次：

```javascript
const ev = JsMacros.createCustomEvent("myScripts:report")
ev.registerEvent() // 之后 GUI 事件下拉框里就能看到它
```

## 监听器清理

监听器不会随脚本结束自动消失——`JsMacros.on` 注册后会一直存在，直到被注销或游戏关闭。不清理就反复运行脚本，会出现"同一条消息打印 N 遍"的经典问题。

### 手动注销

```javascript
const listener = JsMacros.on("Tick", JavaWrapper.methodToJava(() => {}))
// 不需要时:
JsMacros.off(listener)      // 或 listener.off()
```

### 按范围批量关闭

| 方法 | 范围 | 说明 |
| --- | --- | --- |
| `JsMacros.disableScriptListeners()` | 所有**脚本创建**的监听器 | 包括 `on`/`once`/`waitForEvent` 产生的 |
| `JsMacros.disableScriptListeners(event)` | 某事件上的脚本监听器 | 同上，限定单个事件 |
| `JsMacros.disableAllListeners()` | **所有**监听器 | 连 JsMacros 自身（GUI 里绑定）的监听器也会关掉，慎用 |
| `JsMacros.disableAllListeners(event)` | 某事件上的所有监听器 | 同上，限定单个事件 |

!!! warning "disableAllListeners 会误伤"
    `disableAllListeners` 连你在 JsMacros 界面里配置的事件绑定也一并关闭，通常你想要的是 `disableScriptListeners`。

### 服务脚本的正确清理姿势

常驻服务（Service）脚本启动时，全局 `event` 是 `Service` 事件，它提供了停止回调与自动清理：

```javascript
JsMacros.assertEvent(event, "Service")

const listener = JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    // ...
}))
const hud = Hud.createDraw2D()
hud.register()

// 服务停止时自动执行
event.stopListener = JavaWrapper.methodToJava(() => {
    JsMacros.off(listener)
    Chat.log("服务已停止，监听器已清理")
})

// 或者更省事: 停止时自动 off 本服务的监听器，并 unregister 传入的 Draw2D/Draw3D 等
event.unregisterOnStop(true, hud)
```

`unregisterOnStop(offEvents, ...registrables)` 的执行顺序是：`stopListener` → 注销事件监听 → unregister 传入对象 → `postStopListener`。多次调用只保留最后一次设置。详见 [服务](services.md)。

## 事件速查表
[全部事件参考](events_reference.md)。

