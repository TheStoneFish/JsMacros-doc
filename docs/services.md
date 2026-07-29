---
icon: lucide/power
---

# 服务脚本

服务（Service）是 JsMacros 的"常驻后台脚本"：随客户端启动自动运行，长期挂在后台，靠副作用工作——HUD 信息面板、聊天监听、自动重连、后台统计都属于这一类。服务的官方定义是：

> services are background scripts designed to run full time and are mainly noticed by their side effects.
> （服务是设计为全程运行的后台脚本，主要通过副作用体现存在。）

本页讲服务的生命周期、规范写法（重点是**如何正确收尾**）、`ServiceManager` 的完整 API，以及服务专用的全局变量空间。

## 什么是服务

在 JsMacros 主界面的 **Services** 标签页里，你可以把一个脚本文件注册为服务、命名、启用/禁用、手动启停。服务有几个关键特征：

- **常驻**：不是"跑完就退"，而是一直活着，直到被停止；
- **随游戏启动**：已启用（enabled）的服务在游戏启动时自动运行；
- **单实例**：同名服务同时只有一个实例在跑，重复启动无效，改用重启（restart）；
- **专属事件**：服务脚本的全局变量 `event` 是 `Events.Service` 类型，提供停止回调和自动清理能力（这是服务脚本和普通脚本最大的 API 差异）。

## 服务的生命周期

1. **注册**：在 GUI 的 **Services** 标签页添加，或使用 `ServiceManager.registerService(name, path)` 注册。
2. **启用**：启用（Enabled）表示游戏启动时自动启动；启用与运行中（Running）是两个独立状态。
3. **启动**：游戏启动自动启动已启用的服务，或通过 `ServiceManager.startService(name)` 手动启动。
4. **运行**：脚本从头开始执行。默认情况下脚本结束后服务会停止；调用 `event.unregisterOnStop(...)` 后，服务会保持运行，直到手动停止、重新加载脚本或退出游戏。
5. **停止**：通过 `stopService()`、GUI 停止、重新加载脚本或退出游戏触发，并按以下顺序执行：

```
stopListener
    ↓
（可选）注销事件监听（offEvents=true）
    ↓
注销 unregisterOnStop() 登记的对象
    ↓
postStopListener
```

- **stopListener**：停止流程开始时执行，适合保存数据、释放资源等。
- **unregisterOnStop(...)**：自动注销事件监听和 `Registrable` 对象，并使服务在脚本结束后保持运行。
- **postStopListener**：所有自动注销完成后执行，适合最后的收尾工作。

!!! note "文件一改，服务自动重启"
    服务管理器自带"文件变更重载"：服务脚本文件被修改保存后，服务会自动重启，非常适合边写边调。不想要这个行为，可以对单个服务 `disableReload(name)`，或整体 `stopReloadListener()`（恢复用 `startReloadListener()`）。
### stopListener 与 postStopListener

需要在停止时做额外的事（存档、发通知、断开连接），用两个回调字段：

```javascript
JsMacros.assertEvent(event, "Service")

event.stopListener = JavaWrapper.methodToJava(() => {
    Chat.log("服务即将停止, 先把数据写盘...")
    // 此时事件监听还没被注销, 资源还没被 unregister, 自己手动进行注销
    // .......
})

// 整个停止流程的最后一步
event.postStopListener = JavaWrapper.methodToJava(() => {
    Chat.log("收尾完毕") 
})
```

!!! tips "小提示"
    stopListener 和 unregisterOnStop 可以同时使用, 但是好像没有什么必要 = =。
## 规范的服务脚本模板

```javascript
// 一个规范的服务脚本模板: 显示坐标 HUD + 监听聊天
// 断言事件类型, TS 下还能获得补全
JsMacros.assertEvent(event, "Service") 

// 注册资源 
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

// 登记自动清理 
// offEvents = true: 停止时自动注销本服务注册的所有事件监听器(tickListener、chatListener)
// 后面的可变参数: 停止时自动 unregister 的对象(Draw2D、Draw3D、CommandBuilder 等)
event.unregisterOnStop(true, hud)

// 收尾完成后的回调 
event.postStopListener = JavaWrapper.methodToJava(() => {
    Chat.log(`[${event.serviceName}] 已停止, 资源已全部清理`)
})

Chat.log(`[${event.serviceName}] 已启动`)
```

把它保存成 `services/my_hud.js`，在 GUI 的 Services 标签页注册并启动，就是一个可以随意启停、不留垃圾的服务。

!!! warning "常见错误"
    把监听器对象直接塞进参数列表是错的，注销监听靠的是第一个参数 `offEvents = true`。

## 用 ServiceManager 管理服务

入口是 `JsMacros.getServiceManager()`，返回 `ServiceManager`。它让你**用脚本管理其他服务**——比如写一个"看门狗"脚本确保关键服务在线。

### 服务管理方法

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

`status(name)` 把"是否启用"和"是否运行"合成一个值：

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
    // 返回启动前的状态
    const prev = manager.startService(critical) 
    Chat.log(`已拉起 ${critical} (之前状态: ${prev})`)
}
```

用脚本注册一个新服务并持久化：

```javascript
const manager = JsMacros.getServiceManager()
if (manager.registerService("my_hud", "services/my_hud.js", true)) {
    // 写入配置, 下次启动游戏依然存在
    manager.save() 
    manager.startService("my_hud")
} else {
    Chat.log("同名服务已存在")
}
```

## Service 事件的全局变量空间

`Events.Service` 除了停止回调，还带一整套**临时变量空间**, 存储在EventService里面的  `protected Map<String, Object> args = new ConcurrentHashMap<>();` 生命周期就是服务的生命周期。

| 分类 | 方法 | 说明 |
| --- | --- | --- |
| 写入 | `putInt(name, i)` / `putString(name, str)` / `putDouble(name, d)` / `putBoolean(name, b)` / `putObject(name, o)` | 按类型存值 |
| 读取 | `getInt(name)` / `getString(name)` / `getDouble(name)` / `getBoolean(name)` / `getObject(name)` | 按类型取值，不存在返回 `null` |
| 类型 | `getType(name)` | 返回 `'Int'`/`'String'`/`'Double'`/`'Boolean'`/`'Object'` 或 `null` |
| 整数原子操作 | `getAndIncrementInt(name)` / `getAndDecrementInt(name)` / `incrementAndGetInt(name)` / `decrementAndGetInt(name)` | 先取后增减 / 先增减后取 |
| 布尔 | `toggleBoolean(name)` | 翻转并返回新值 |
| 管理 | `remove(key)` / `getRaw()` | 删除键 / 拿到底层 `JavaMap<string, any>` |

计数器实例-跨脚本通信：

```javascript
JsMacros.assertEvent(event, 'Service')
let count = 0
while (true) {
    count++
    event.putInt('count', count)
    Chat.log(`计数器更新: ${count}`)
    Client.waitTick(20)
}
```

其他脚本可以读到数据：

```javascript
// 获取smr管理服务
const smr = JsMacros.getServiceManager()
// 这里需要为服务名称 --> 实际获取的值就是eventService变量
const exampleEventService = smr.getServiceData('example')
const count = exampleEventService.getInt('count')
Chat.log(`example 计时器值为: ${count}`)
```

!!! tip "变量是全局的吗?"
    变量存储在当前 **EventService 对象** 的数据空间中。
    同一个服务中的其他脚本可以通过 `ServiceManager.getServiceData(name)` 获取该 `EventService` 对象，并读取其中的数据。

## 轮询循环的正确写法

服务里能用事件监听就不要 `while(true)`（轮询 Tick 请直接 `JsMacros.on("Tick", ...)`, 确实要写循环时，**必须有退出条件**，并且每轮交出控制权：

```javascript
JsMacros.assertEvent(event, "Service")

let running = true
event.stopListener = JavaWrapper.methodToJava(() => {
    // 通知循环优雅退出
    running = false
})

while (running) {
    // ...每轮要做的事...
    // 交出控制权, 也给 stopListener 执行的机会
    Client.waitTick() 
}
Chat.log("循环退出, 服务结束")
```

!!! note "就算你不写退出条件"
    停止服务时脚本上下文会被关闭，正在 `waitTick()`/`Time.sleep()` 的线程会被中断（抛 `InterruptedException`），服务通常还是能停下来——但那是"被杀"，你的收尾代码没机会跑完。退出标志才是优雅关闭。
