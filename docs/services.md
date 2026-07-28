---
icon: lucide/power
---

# 服务脚本

服务（Service）是 JsMacros 的"常驻后台脚本"：随客户端启动自动运行，长期挂在后台，靠副作用工作——HUD 信息面板、聊天监听、自动重连、后台统计都属于这一类。d.ts 里对服务的官方定义是：

> services are background scripts designed to run full time and are mainly noticed by their side effects.
> （服务是设计为全程运行的后台脚本，主要通过副作用体现存在。）

本页讲服务的生命周期、规范写法（重点是**如何正确收尾**）、`ServiceManager` 的完整 API，以及服务专用的全局变量空间。

## 什么是服务

在 JsMacros 主界面的 **Services** 标签页里，你可以把一个脚本文件注册为服务、命名、启用/禁用、手动启停。服务有几个关键特征：

- **常驻**：不是"跑完就退"，而是一直活着，直到被停止；
- **随游戏启动**：已启用（enabled）的服务在游戏启动时自动运行；
- **单实例**：同名服务同时只有一个实例在跑，重复启动无效，改用重启（restart）；
- **专属事件**：服务脚本的全局变量 `event` 是 `Events.Service` 类型，提供停止回调和自动清理能力（这是服务脚本和普通脚本最大的 API 差异）。

## 服务 vs 按键/事件宏

| | 按键/事件宏 | 服务 |
| --- | --- | --- |
| 启动方式 | 按键、事件触发 | 游戏启动自动 / GUI 手动 / `ServiceManager` |
| 生命周期 | 跑完即走（除非挂了监听器） | 常驻后台，直到被停止 |
| 实例 | 每次触发新开一个线程实例 | 同名服务只有一个实例 |
| 全局 `event` | `Events.Key`、`Events.RecvMessage` 等 | `Events.Service` |
| 停止 | 自然结束 / GUI 停止 | `stopService` / GUI，停止时有收尾回调 |
| 典型用途 | 一键宏、临时操作 | HUD、聊天监听、自动重连、后台统计 |

!!! tip "怎么选"
    只要脚本里出现了 `JsMacros.on(...)` 常驻监听或者注册了 HUD，它本质上就是常驻脚本——那就把它做成服务，享受服务的启停管理和自动清理，而不是靠按键宏"跑一次挂一堆监听器"。

## 服务的生命周期

1. **注册**：在 GUI 的 Services 标签页添加，或用 `ServiceManager.registerService(name, path)`（路径相对宏文件夹）；
2. **启用/禁用**：启用（enabled）表示随游戏启动自动运行，会写进配置；禁用则只能手动启动。启用与"正在运行"是两个独立状态；
3. **启动**：游戏启动自动拉起已启用的服务，或 `startService(name)` 手动启动；
4. **运行**：脚本从头执行；只要设置过 `unregisterOnStop`，即使代码跑到最后一行服务也不会自行结束（d.ts 原文：*If anything was set to unregister, the service won't stop by itself even if it reaches the end*）；
5. **停止**：`stopService(name)`、GUI 停止或游戏退出，触发四步收尾流程（顺序来自 d.ts）：

```
执行 stopListener → 注销事件监听（offEvents 为 true 时）→ unregister 登记的对象 → 执行 postStopListener
```

!!! note "文件一改，服务自动重启"
    2.1.0 的服务管理器自带"文件变更重载"：服务脚本文件被修改保存后，服务会自动重启，非常适合边写边调。不想要这个行为，可以对单个服务 `disableReload(name)`，或整体 `stopReloadListener()`（恢复用 `startReloadListener()`）。

## 规范的服务脚本模板

这是本页最重要的部分。一个规范的服务脚本 = **注册资源 + 登记自动清理 + （可选）收尾回调**，三件事缺一不可，否则停止服务后会留下"幽灵监听器"和残留 HUD。

```javascript
// 一个规范的服务脚本模板: 显示坐标 HUD + 监听聊天
JsMacros.assertEvent(event, "Service") // 断言事件类型, TS 下还能获得补全

// ---- 1. 注册资源 ----
const hud = Hud.createDraw2D()
const text = hud.addText("服务启动中...", 10, 10, 0xFFFFFF, true)
hud.register()

const tickListener = JsMacros.on("Tick", JsMacros.eventFilters().modulus(10),
    JavaWrapper.methodToJava(() => {
        const p = Player.getPlayer()
        if (p) text.setText(`X=${p.getX().toFixed(1)} Y=${p.getY().toFixed(1)} Z=${p.getZ().toFixed(1)}`)
    }))

const chatListener = JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    if (e.text && e.text.getString().includes("@" + Player.getPlayer().getName().getString())) {
        Chat.log("§e有人@你了!")
    }
}))

// ---- 2. 登记自动清理 ----
// offEvents = true: 停止时自动注销本服务注册的所有事件监听器(tickListener、chatListener)
// 后面的可变参数: 停止时自动 unregister 的对象(Draw2D、Draw3D、CommandBuilder 等)
event.unregisterOnStop(true, hud)

// ---- 3. (可选)收尾完成后的回调 ----
event.postStopListener = JavaWrapper.methodToJava(() => {
    Chat.log(`[${event.serviceName}] 已停止, 资源已全部清理`)
})

Chat.log(`[${event.serviceName}] 已启动`)
```

把它保存成 `services/my_hud.js`，在 GUI 的 Services 标签页注册并启动，就是一个可以随意启停、不留垃圾的服务。

关于 `unregisterOnStop(offEvents, ...list)` 的几个细节（均来自 d.ts JSDoc）：

- `offEvents` 为 `true` 时，服务管理器会在停止时清掉本上下文注册的事件监听器——所以监听器**不需要**（也不能）塞进后面的参数列表，那里只收 `Registrable` 对象（Draw2D、Draw3D、CommandBuilder 等）；
- **多次调用只保留最后一次**，把要清理的对象一次性传全；
- `event.unregisterOnStop(false, d2d)` 的效果等价于 `event.stopListener = JavaWrapper.methodToJava(() => d2d.unregister())`；
- 调用过它之后，脚本跑到末尾服务也会保持存活——这正是服务想要的。

!!! warning "旧教程的常见错误"
    网上一些旧例子写 `context.unregisterOnStop(...)`——这个方法在 `event`（`Events.Service`）上，不在 `context` 上；另外把监听器对象直接塞进参数列表也是错的，注销监听靠的是第一个参数 `offEvents = true`。

### 手动收尾：stopListener 与 postStopListener

需要在停止时做额外的事（存档、发通知、断开连接），用两个回调字段：

```javascript
JsMacros.assertEvent(event, "Service")

event.stopListener = JavaWrapper.methodToJava(() => {
    Chat.log("服务即将停止, 先把数据写盘...")
    // 此时事件监听还没被注销, 资源还没被 unregister
})

event.postStopListener = JavaWrapper.methodToJava(() => {
    Chat.log("收尾完毕") // 整个停止流程的最后一步
})
```

!!! warning "stopListener 和 unregisterOnStop 二选一"
    d.ts 明确说 `unregisterOnStop` 等价于设置 `stopListener`，且重复设置会相互覆盖。所以：**要么**自己写 `stopListener` 手动清理一切，**要么**用一次 `unregisterOnStop` 全权托管；两条路都想走的话，把额外收尾逻辑放进 `postStopListener`（它是独立字段，始终在流程最后执行）。

## 用 ServiceManager 管理服务

入口是 `JsMacros.getServiceManager()`，返回 `ServiceManager`。它让你**用脚本管理其他服务**——比如写一个"看门狗"脚本确保关键服务在线。

### 方法全表

注册与配置：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `registerService(name, pathToFile)` | `boolean` | 注册服务，路径相对宏文件夹；重名返回 `false` |
| `registerService(name, pathToFile, enabled)` | `boolean` | 注册并指定是否启用 |
| `registerService(name, trigger)` | `boolean` | 用 `ServiceTrigger` 对象注册 |
| `unregisterService(name)` | `boolean` | 删除服务 |
| `renameService(oldName, newName)` | `boolean` | 重命名；新名已存在或旧名不存在返回 `false` |
| `getServices()` | `JavaSet<string>` | 全部已注册服务名 |
| `load()` | `void` | 从配置加载服务 |
| `save()` | `void` | 把当前注册的服务及启用状态存入配置 |

启动与停止（这组方法都返回操作**之前**的 `ServiceStatus`，未知服务返回 `UNKNOWN`）：

| 方法 | 说明 |
| --- | --- |
| `startService(name)` | 启动一次 |
| `stopService(name)` | 停止 |
| `restartService(name)` | 重启 |
| `enableService(name)` | 启用（随游戏启动自动运行） |
| `disableService(name)` | 禁用 |

状态查询：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isRunning(name)` | `boolean` | 是否正在运行 |
| `isEnabled(name)` | `boolean` | 是否已启用 |
| `status(name)` | `ServiceStatus` | 综合状态（见下表） |
| `getServiceData(name)` | `Events.Service` | 该服务当前的 Service 事件对象；**服务没在运行时可能抛异常** |
| `getTrigger(name)` | `ServiceTrigger` | 服务的触发器配置（文件 + 是否启用） |

文件变更重载相关：

| 方法 | 说明 |
| --- | --- |
| `startReloadListener()` / `stopReloadListener()` | 开启/停止"文件变更自动重载" |
| `disableReload(serviceName)` | 关闭单个服务的自动重载 |
| `tickReloadListener()` | 手动驱动一次重载检查（内部机制，一般用不到） |
| `markCrashed(serviceName)` / `isCrashed(serviceName)` | 标记/查询服务崩溃状态，崩溃的服务在文件变更时也会被重启 |

另有两个静态方法 `ServiceManager.setAutoUnregisterKeepAlive(ctx, keepAlive)` 和 `hasKeepAlive(ctx)`，控制脚本上下文的保活标记，属内部机制，日常脚本用不到。

### ServiceStatus 枚举

`status(name)` 把"是否启用"和"是否运行"合成一个值（含义来自 d.ts JSDoc）：

| 值 | 已启用? | 在运行? |
| --- | --- | --- |
| `ENABLED` | 是 | 是 |
| `STOPPED` | 是 | 否 |
| `RUNNING` | 否 | 是 |
| `DISABLED` | 否 | 否 |
| `UNKNOWN` | 服务不存在 | — |

### ServiceTrigger

`getTrigger(name)` 返回的 `ServiceTrigger` 只有两个字段和一个方法：`file`（`java.nio.file.Path`，脚本文件）、`enabled`（是否启用）、`toScriptTrigger()`（转换成通用的 `ScriptTrigger`）。

### 示例：服务面板与看门狗

列出所有服务和状态：

```javascript
const manager = JsMacros.getServiceManager()
Chat.log("=== 服务列表 ===")
for (const name of manager.getServices()) {
    Chat.log(`${name}: ${manager.status(name)}` +
        ` (运行=${manager.isRunning(name)}, 启用=${manager.isEnabled(name)})`)
}
```

确保关键服务在线（可以本身做成服务，定期检查）：

```javascript
const manager = JsMacros.getServiceManager()
const critical = "chat_logger"

if (manager.getServices().contains(critical) && !manager.isRunning(critical)) {
    const prev = manager.startService(critical) // 返回启动前的状态
    Chat.log(`已拉起 ${critical} (之前状态: ${prev})`)
}
```

用脚本注册一个新服务并持久化：

```javascript
const manager = JsMacros.getServiceManager()
if (manager.registerService("my_hud", "services/my_hud.js", true)) {
    manager.save() // 写入配置, 下次启动游戏依然存在
    manager.startService("my_hud")
} else {
    Chat.log("同名服务已存在")
}
```

## Service 事件的全局变量空间

`Events.Service` 除了停止回调，还带一整套**全局变量空间**读写方法（共 18 个）。它和 `GlobalVars` 库操作的是**同一个空间**（d.ts 里两者 JSDoc 措辞完全一致）：数据在整个游戏会话期间一直存在，**服务停了再启、脚本重载都不清空**，游戏关闭才消失——这正是服务保存"跨启停状态"的标准位置。

| 分类 | 方法 | 说明 |
| --- | --- | --- |
| 写入 | `putInt(name, i)` / `putString(name, str)` / `putDouble(name, d)` / `putBoolean(name, b)` / `putObject(name, o)` | 按类型存值 |
| 读取 | `getInt(name)` / `getString(name)` / `getDouble(name)` / `getBoolean(name)` / `getObject(name)` | 按类型取值，不存在返回 `null` |
| 类型 | `getType(name)` | 返回 `'Int'`/`'String'`/`'Double'`/`'Boolean'`/`'Object'` 或 `null` |
| 整数原子操作 | `getAndIncrementInt(name)` / `getAndDecrementInt(name)` / `incrementAndGetInt(name)` / `decrementAndGetInt(name)` | 先取后增减 / 先增减后取 |
| 布尔 | `toggleBoolean(name)` | 翻转并返回新值 |
| 管理 | `remove(key)` / `getRaw()` | 删除键 / 拿到底层 `JavaMap<string, any>` |

跨启停示例——统计服务本次游戏会话里被（重）启动了几次：

```javascript
JsMacros.assertEvent(event, "Service")

const KEY = `${event.serviceName}:starts` // 键名加服务前缀, 避免和别的服务撞车
if (event.getInt(KEY) === null) event.putInt(KEY, 0)

const n = event.incrementAndGetInt(KEY)
Chat.log(`[${event.serviceName}] 本会话第 ${n} 次启动`)
```

其他脚本可以直接用 `GlobalVars` 读到同一份数据：

```javascript
Chat.log(`my_hud 启动次数: ${GlobalVars.getInt("my_hud:starts")}`)
```

!!! tip "要跨游戏重启? 写文件"
    全局变量空间只活一个游戏会话。需要真正持久化的配置，用 `FS` 存 JSON：

    ```javascript
    const configPath = "my_service/config.json"
    if (!FS.exists(configPath)) {
        FS.createFile("my_service", "config.json", true) // 第三个参数: 自动创建目录
        FS.open(configPath).write(JSON.stringify({ enabled: true }, null, 2))
    }
    const config = JSON.parse(FS.open(configPath).read())
    ```

## 常见坑

| 坑 | 处理 |
| --- | --- |
| 停止服务后 HUD 还在 | `event.unregisterOnStop(true, hud)`，注意是 `event` 不是 `context` |
| 停止服务后监听器还在触发 | `unregisterOnStop` 第一个参数传 `true`；或在 `stopListener` 里手动 `JsMacros.off(listener)` |
| 改文件后监听器越叠越多 | 文件变更会自动**重启**服务，只要做好了停止清理就不会叠加；没做清理才会残留 |
| `while(true)` 停不下来 | 见下方"轮询循环的正确写法" |
| 多个服务共享状态互相覆盖 | 全局变量空间键名加服务前缀，如 `"my_hud:xxx"` |
| `getServiceData(name)` 抛异常 | 服务没在运行时可能抛，先 `isRunning(name)` 判断 |

### 轮询循环的正确写法

服务里能用事件监听就不要 `while(true)`（轮询 Tick 请直接 `JsMacros.on("Tick", ...)` 配合[事件过滤器](event_filters.md)的 `modulus` 降频）。确实要写循环时，**必须有退出条件**，并且每轮交出控制权：

```javascript
JsMacros.assertEvent(event, "Service")

let running = true
event.stopListener = JavaWrapper.methodToJava(() => {
    running = false // 通知循环优雅退出
})

while (running) {
    // ...每轮要做的事...
    Client.waitTick() // 交出控制权, 也给 stopListener 执行的机会
}
Chat.log("循环退出, 服务结束")
```

!!! note "就算你不写退出条件"
    停止服务时脚本上下文会被关闭，正在 `waitTick()`/`Time.sleep()` 的线程会被中断（抛 `InterruptedException`），服务通常还是能停下来——但那是"被杀"，你的收尾代码没机会跑完。退出标志才是优雅关闭。
