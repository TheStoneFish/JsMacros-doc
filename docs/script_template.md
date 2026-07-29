---
icon: lucide/file-code-2
---

# 脚本模板

很多 JsMacros 脚本不是"运行一次就结束"，而是会循环、监听事件、等容器打开、等寻路、等聊天返回。新手最容易踩的坑是：你以为脚本关了，其实旧线程还在 `Time.sleep()` 里面睡觉。

这一页给出一个"按一次开、再按一次关"的启停模板，然后逐段讲清楚每一块为什么这么写。

## 按键型脚本模板

把它绑到一个按键上（fork 状态），按一次开启，再按一次关闭：
#### 完整代码
```javascript
// 每个脚本改成自己的名字，不要重复
const scriptName = 'EXAMPLE'

GlobalVars.toggleBoolean(scriptName)

function isEnabled() {
    return GlobalVars.getBoolean(scriptName)
}

if (isEnabled()) {
    Chat.log(`§7[§b${scriptName}§7] §a已开启`)
} else {
    Chat.log(`§7[§b${scriptName}§7] §c已关闭`)
}

// 日志分级
const LOG_LEVELS = { debug: 0, info: 1, warn: 2, error: 3 }
// 排查问题时改成 LOG_LEVELS.debug
let currentLogLevel = LOG_LEVELS.info

// 监听器
const listeners = []
function log(msg, level = 'info') {
    const lvl = LOG_LEVELS[level] ?? LOG_LEVELS.info
    if (lvl < currentLogLevel) return

    let color = '§7'
    if (level === 'debug') color = '§8'
    if (level === 'warn') color = '§e'
    if (level === 'error') color = '§c'

    Chat.log(`§7[§b${scriptName}§7] ${color}[${level.toUpperCase()}]§r ${msg}`)
}

function addListener(eventName, callback) {
    const listener = JsMacros.on(eventName, JavaWrapper.methodToJava(callback))
    listeners.push({ name: eventName, listener })
    return listener
}
// 安全睡眠
function safeSleep(ms, step = 50) {
    const end = Time.time() + ms
    while (isEnabled() && Time.time() < end) {
        Time.sleep(Math.min(step, end - Time.time()))
    }
    return isEnabled()
}

function handleRecvMessage(e) {
    log(`收到消息: ${e.text.getString()}`, 'debug')
}

function init() {
    if (!isEnabled()) {
        return
    }
    // 监听器注册区
    addListener('RecvMessage', handleRecvMessage)
    // 执行主循环
    main()
}

function main() {
    // 主循环
    while (true) {
        if (!isEnabled()) {
            for (const { name, listener } of listeners) {
                log(`移除监听器 ${name}`)
                JsMacros.off(listener)
            }
            listeners.splice(0)
            break
        }
        if (!World.isWorldLoaded()) {
            Client.waitTick(1)
            continue
        }
        if (!Player.getPlayer()) {
            Client.waitTick(1)
            continue
        }

        // 主逻辑写这里
        log('Hello World', 'debug')
        safeSleep(200)
    }
}
init()

```
#### 全局变量控制

```javascript
GlobalVars.toggleBoolean(scriptName)
```

`GlobalVars` 是**所有脚本共享**的一块内存(Java MAP)（详见[全局变量共享](globals.md)）。`toggleBoolean(name)` 把里面的一个布尔值取反并返回新值。

按键每按一次，就是**一次全新的脚本运行**。于是整个流程变成：

1. 第一次运行：键还不存在，取反后变成 `true`，本次运行进入主循环，成为"工作线程"。
2. 第二次运行：取反成 `false`，本次运行走关闭分支，几乎立刻结束；而上一次运行的主循环在下一次检查 `isEnabled()` 时发现开关关了，自己退出。

!!! note "关键理解"
    "关闭脚本"不是新一次运行把旧线程杀掉，而是它把共享开关拨到 `false`，旧线程**自己看见后退出**。所以旧线程里每一个循环、每一次长等待，都必须有机会看到这个开关。

#### isEnabled 为什么是函数不是变量

```javascript
function isEnabled() {
    return GlobalVars.getBoolean(scriptName)
}
```

如果写成 `const enabled = GlobalVars.getBoolean(scriptName)`，取出来的只是运行那一刻的快照，之后再也感知不到变化。必须每次都去 `GlobalVars` 现查，开关才能"实时生效"。

#### 日志分级系统

```javascript
// info，平时可见
log("准备就绪")
// debug，默认不显示            
log("循环第 3 次", "debug") 
log("背包快满了", "warn")
log("找不到目标容器", "error")
```

比到处写 `Chat.log` 好在两点：

- **一键静音**：调试完把 `currentLogLevel` 改回 `LOG_LEVELS.info`，所有 `debug` 日志立刻消失，不用一行行删；
- **统一前缀和颜色**：多脚本同时运行时，一眼看出消息是谁发的。


#### 安全睡眠

先看坏例子：

```javascript
while (GlobalVars.getBoolean("bad_script")) {
    Chat.log("开始等待")
    Time.sleep(10000)
    Chat.log("等待结束，继续执行")
}
```

`Time.sleep(10000)` 期间线程完全睡死，**不会检查开关**。你按键关闭脚本后，旧线程还会睡满 10 秒；醒来后 `sleep` 后面那句照样执行，下一轮循环才退出。体验就是"我明明关了，它还在动"。

`safeSleep` 的思路是把长睡拆成很多小段，每段之间看一眼开关：

```javascript
function safeSleep(ms, step = 50) {
    const end = Time.time() + ms
    while (isEnabled() && Time.time() < end) {
        Time.sleep(Math.min(step, end - Time.time()))
    }
    return isEnabled()
}
```

- 脚本仍开启 → 继续睡，最多误差一个 `step`（50ms）；
- 脚本被关闭 → 最多 50ms 内醒来，返回 `false`。

调用时**要接返回值**，发现关闭立刻收工：

```javascript
// 在循环里
if (!safeSleep(5000)) break
// 在函数里
if (!safeSleep(5000)) return
```

#### 监听器注册区为什么包在 if (isEnabled()) 里

关闭那次运行也会从头到尾执行整个文件。如果注册语句裸露在外面，"关闭"的那次运行反而又注册了一份新监听器。所以：

- 注册语句包在 `if (isEnabled()) { ... }` 里，只有开启那轮才注册；

#### 主循环的判空

```javascript
if (!World.isWorldLoaded()) { Client.waitTick(1); continue }
if (!Player.getPlayer()) { Client.waitTick(1); continue }
```

脚本可能在主菜单、加载界面、切维度的瞬间运行，此时 `World.isWorldLoaded()` 为 `false`、`Player.getPlayer()` 为 `null`，直接调用玩家方法会报错。判空失败时用 `Client.waitTick(1)` 等下一个游戏 tick 再试。


## 异步任务也要看 isEnabled

```javascript
JavaWrapper.methodToJavaAsync(() => {
    while (true) {
        Time.sleep(200)
    }
}).run()
```

这是一个很危险的孤儿线程：没有任何退出条件，脚本关了它还在跑。应该写成：

```javascript
JavaWrapper.methodToJavaAsync(() => {
    while (isEnabled()) {
        // 后台检查逻辑
        if (!safeSleep(200)) return
    }
}).run()
```

!!! note "关于 .run()"
    `MethodWrapper` 在 Java 侧实现了 `Runnable` 等一系列函数式接口，所以可以 `.run()` 启动


## 进阶：session 防止旧线程复活

boolean 开关有一个极端场景处理不了——**快速关了又开**：

1. 旧线程正在 `safeSleep` 的某个小段里睡 50ms；
2. 你运行脚本关闭（boolean 变 `false`）；
3. 又立刻运行脚本开启（boolean 变回 `true`）；
4. 旧线程醒来一看，开关是开的，继续跑；
5. 旧线程 + 新线程同时工作，逻辑翻倍。

解决办法是给每一轮运行发一个"届号"，旧线程发现届号过期就退出：

```javascript
const KEY_SESSION = `${scriptName}.session`
if (GlobalVars.getType(KEY_SESSION) !== "Int") {
    GlobalVars.putInt(KEY_SESSION, 0)
}
const session = GlobalVars.incrementAndGetInt(KEY_SESSION)

function isEnabled() {
    return GlobalVars.getBoolean(scriptName) === true
        && GlobalVars.getInt(KEY_SESSION) === session
}
```
每次运行都会让 `KEY_SESSION` 自增，只有"最新一届"的 `isEnabled()` 才为真。日常脚本用不用 session 都行；一旦你发现"关了再开会出现双倍消息"，就是时候加上它了。

## 进阶：使用事件发布/订阅来控制开关
``` javascript

// 定义全局变量
let Killaura = false
function init() {
    // 创建一个自定义事件
    const KillauraEvent = JsMacros.createCustomEvent('KillauraEvent')
    KillauraEvent.registerEvent()
    const profile = JsMacros.getProfile()
    const listeners = profile.getRegistry().getListeners('KillauraEvent')
    // 检查目前是否有事件在运行
    if (listeners.size() > 0) {
        // 触发事件 用于关闭之前的脚本
        KillauraEvent.trigger()
        Killaura = false
        Chat.actionbar('§c杀戮光环已关闭')
    } else {
        // 监听事件, 用于监听关闭脚本的信号
        JsMacros.once(
            'KillauraEvent',
            JavaWrapper.methodToJava((e) => {
                Killaura = false
            })
        )
        Killaura = true
        Chat.actionbar('§a杀戮光环已开启')
    }
}
init()
while (Killaura) {
    Client.waitTick(1)

    const entities = World.getEntities(3).filter(
        (e) => e.isAlive() && e.getType() === 'minecraft:zombie'
    )
    if (entities.length === 0) continue

    const player = Player.getPlayer()
    if (player.getAttackCooldownProgress() !== 1) continue

    const nearestEntity = entities.reduce((prev, curr) => {
        return player.distanceTo(curr) < player.distanceTo(prev) ? curr : prev
    })

    player.attack(nearestEntity)
}

```
## 进阶：快速停止脚本
在某些场景需要快速停止脚本（例如发现附近有玩家）。这时 `JavaWrapper.stop()` 是更好的选择。

!!! tip "为什么不靠到处检测开关？"

    开关必须在每一层循环、每一次等待里都判断一次。嵌套一深（寻路 → 开箱 → 挪物品 → 再等待），到处写 `if (!isEnabled()) return` 又烦又容易漏。

    `JavaWrapper.stop()` 直接结束当前脚本上下文，适合「一检测到就立刻停」，而不必在每个分支里重复检查开关。

## 服务型脚本模板

常驻脚本更适合做成[服务](services.md)：随游戏启动自动运行，由服务管理器统一开关，不再需要 `GlobalVars` 开关。服务脚本里 `event` 是 `Service` 事件，专门提供了停止时的清理钩子：

```javascript
// 判断必须是服务因为使用了unregisterOnStop
JsMacros.assertEvent(event, 'Service')

const draw = Hud.createDraw2D()
draw.setOnInit(JavaWrapper.methodToJava(() => {
    draw.addText("服务运行中", 10, 10, 0xFFFFFF, true)
}))
draw.register()

const listener = JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    // ...
}))

// 服务停止时：自动注销 draw，并清理本上下文的监听器(上面的RecvMessage)
event.unregisterOnStop(true, draw)
```

`event.unregisterOnStop(offEvents, ...registrables)` 会在服务停止时按顺序执行清理；设置过它之后，服务即使跑到文件末尾也不会自动结束 等同于`event.stopListener = JavaWrapper.methodToJava(() => draw.unregister())`。细节见[服务](services.md)。

