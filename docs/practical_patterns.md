---
icon: lucide/wrench
---

# 实战模式

这一页收一些真实脚本里反复出现的通用写法。它不绑定某个玩法，只讲脚本怎么写得稳一点。

## 开关脚本

同一个脚本运行一次开启，再运行一次关闭。

```javascript
const scriptName = "my_script"
const enabled = GlobalVars.toggleBoolean(scriptName)

Chat.log(enabled ? "脚本已开启" : "脚本已关闭")

if (!enabled) {
    JsMacros.disableScriptListeners()
    return
}

while (GlobalVars.getBoolean(scriptName)) {
    // 主逻辑
    Client.waitTick(1)
}
```

适合：自动钓鱼、HUD、后台扫描、辅助菜单。

## 世界和玩家判空

很多脚本崩在“还没进世界”或“切维度瞬间玩家为空”。

```javascript
function ready() {
    return World.isWorldLoaded() && Player.getPlayer()
}

while (GlobalVars.getBoolean("my_script")) {
    if (!ready()) {
        Client.waitTick(1)
        continue
    }

    const player = Player.getPlayer()
    Chat.actionbar(`Y=${player.getY().toFixed(1)}`)
    Client.waitTick(20)
}
```

事件里也一样：

```javascript
JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    if (!World.isWorldLoaded() || !Player.getPlayer()) return
    Chat.log(e.text.getString())
}))
```

## 安全睡眠

`Time.sleep()` 不知道你的脚本是否已经关闭。长等待建议拆成小段。

```javascript
function safeSleep(ms, key = "my_script") {
    const end = Time.time() + ms
    while (Time.time() < end) {
        if (!GlobalVars.getBoolean(key)) return false
        Time.sleep(Math.min(100, end - Time.time()))
    }
    return true
}
```

## 后台工作线程

某些工作不想卡住主流程，可以用异步包装再 `.run()`。

```javascript
JavaWrapper.methodToJavaAsync(() => {
    while (GlobalVars.getBoolean("my_script")) {
        // 后台检查
        Time.sleep(200)
    }
}).run()
```

!!! warning "别忘了退出条件"
    异步循环必须有开关或停止条件，不然脚本关了它还在跑。

## 等待界面打开

容器脚本不要点太快。先等标题或容器状态。

```javascript
function waitContainerTitle(name, timeoutMs = 3000) {
    const start = Time.time()
    while (Time.time() - start < timeoutMs) {
        if (!Player.getPlayer()) return false

        const inv = Player.openInventory()
        if (inv.isContainer() && inv.getContainerTitle().includes(name)) {
            return true
        }

        Time.sleep(50)
    }

    return false
}

if (waitContainerTitle("确认购买")) {
    Player.openInventory().click(11)
}
```

## 容器操作节流

```javascript
const inv = Player.openInventory()
inv.click(13)
Client.waitTick(2)
inv.quick(20)
Client.waitTick(2)
```

连续点击容器时，加 `Client.waitTick(1)` 或 `Client.waitTick(2)` 能减少客户端/服务器不同步。

## 配置文件

```javascript
const configPath = "my_script/config.json"

function loadConfig() {
    if (!FS.exists("my_script")) FS.makeDir("my_script")
    if (!FS.exists(configPath)) {
        FS.open(configPath).write(JSON.stringify({
            enabled: true,
            delay: 200,
        }, null, 2))
    }

    return JSON.parse(FS.open(configPath).read())
}

const config = loadConfig()
```

## 日志文件

```javascript
function appendLog(message) {
    if (!FS.exists("logs")) FS.makeDir("logs")
    FS.open("logs/my_script.log").append(`[${new Date().toLocaleString()}] ${message}\n`)
}
```

!!! tip "append，不是 write"
    追加日志用 `append()`。`write()` 会覆盖整个文件。

## HUD 清理

```javascript
const draw = Hud.createDraw2D()
draw.addText("running", 10, 10, 0xFFFFFF, true)
draw.register()

try {
    while (GlobalVars.getBoolean("my_script")) {
        Client.waitTick(20)
    }
} finally {
    draw.unregister()
}
```

服务脚本可以用：

```javascript
context.unregisterOnStop(true, draw)
```

## 事件监听清理

```javascript
const listener = JsMacros.on("Tick", JavaWrapper.methodToJava(() => {
    // ...
}))

context.unregisterOnStop(true, listener)
```

开关脚本关闭时也可以：

```javascript
JsMacros.disableScriptListeners()
```

## 超时模式

任何等待外部状态的循环都应该有超时。

```javascript
function waitUntil(check, timeoutMs = 5000) {
    const start = Time.time()
    while (Time.time() - start < timeoutMs) {
        if (check()) return true
        Time.sleep(50)
    }
    return false
}
```

适合等待：容器打开、寻路结束、聊天返回、物品刷新、世界加载。

