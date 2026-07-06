---
icon: lucide/power
---

# 服务脚本

服务脚本适合长期运行：HUD、聊天监听、自动重连、后台状态维护。和普通按键宏相比，服务通常随客户端启动或手动启停，不能重复开多个实例。

## 服务脚本的常见结构

```javascript
const draw = Hud.createDraw2D()
const text = draw.addText("service running", 10, 10, 0xFFFFFF, true)
draw.register()

const listener = JsMacros.on("Tick", JavaWrapper.methodToJava(() => {
    const p = Player.getPlayer()
    if (p) text.setText(`Y=${p.getY().toFixed(1)}`)
}))

context.unregisterOnStop(true, draw, listener)
```

`unregisterOnStop(true, ...)` 会在服务停止时注销注册对象和事件监听。

## stopListener 和 postStopListener

服务事件 `event` 有停止回调：

```javascript
event.stopListener = JavaWrapper.methodToJava(() => {
    Chat.log("服务即将停止")
})

event.postStopListener = JavaWrapper.methodToJava(() => {
    Chat.log("服务已停止")
})
```

停止顺序大致是：

1. 执行 `stopListener`
2. 移除事件监听
3. 注销通过 `context.unregisterOnStop` 登记的对象
4. 执行 `postStopListener`

## ServiceManager

```javascript
const manager = JsMacros.getServiceManager()
Chat.log(manager.getServices())
```

常用方法：

| 方法 | 作用 |
| --- | --- |
| `registerService(name, pathToFile)` | 注册服务 |
| `unregisterService(name)` | 删除服务 |
| `startService(name)` | 启动 |
| `stopService(name)` | 停止 |
| `restartService(name)` | 重启 |
| `enableService(name)` / `disableService(name)` | 启用/禁用 |
| `isRunning(name)` | 是否运行 |
| `isEnabled(name)` | 是否启用 |
| `status(name)` | 状态 |
| `getServiceData(name)` | 取得服务事件数据 |
| `save()` / `load()` | 保存/加载配置 |

## 服务里保存状态

简单状态可以用 `GlobalVars`：

```javascript
GlobalVars.putInt("my_service.ticks", 0)

JsMacros.on("Tick", JavaWrapper.methodToJava(() => {
    GlobalVars.incrementAndGetInt("my_service.ticks")
}))
```

长期配置用 `FS` 存 JSON：

```javascript
const configPath = "my_service/config.json"
if (!FS.exists(configPath)) {
    FS.createFile("my_service", "config.json", true)
    FS.open(configPath).write(JSON.stringify({ enabled: true }, null, 2))
}
```

## 常见坑

| 坑 | 处理 |
| --- | --- |
| 停止服务后 HUD 还在 | 用 `context.unregisterOnStop(true, draw)` |
| 重载后监听器叠加 | 停止时注销 listener，或启动前 `JsMacros.disableScriptListeners()` |
| 服务里死循环太猛 | 循环内 `Client.waitTick()` |
| 多服务共享状态冲突 | `GlobalVars` 键名加服务前缀 |

