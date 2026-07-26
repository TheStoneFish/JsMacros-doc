---
icon: lucide/file-code-2
---

# 脚本模板

很多 JsMacros 脚本不是"运行一次就结束"，而是会循环、监听事件、等容器打开、等寻路、等聊天返回。新手最容易踩的坑是：你以为脚本关了，其实旧线程还在 `Time.sleep()` 里面睡觉。

这一页给出一个"按一次开、再按一次关"的启停模板，然后逐段讲清楚每一块为什么这么写。

## 完整模板

把它绑到一个按键上（fork 状态），按一次开启，再按一次关闭：

```javascript
// ================= 基本信息 =================
const scriptName = "EXAMPLE" // 每个脚本改成自己的名字，不要重复

// ================= 开关 =================
GlobalVars.toggleBoolean(scriptName)

function isEnabled() {
    return GlobalVars.getBoolean(scriptName)
}

if (isEnabled()) {
    Chat.log(`§7[§b${scriptName}§7] §a已开启`)
} else {
    Chat.log(`§7[§b${scriptName}§7] §c已关闭`)
    JsMacros.disableScriptListeners()
}

// ================= 日志分级 =================
const LOG_LEVELS = { debug: 0, info: 1, warn: 2, error: 3 }
let currentLogLevel = LOG_LEVELS.info // 排查问题时改成 LOG_LEVELS.debug

function log(msg, level = "info") {
    const lvl = LOG_LEVELS[level] ?? LOG_LEVELS.info
    if (lvl < currentLogLevel) return

    let color = "§7"
    if (level === "debug") color = "§8"
    if (level === "warn") color = "§e"
    if (level === "error") color = "§c"

    Chat.log(`§7[§b${scriptName}§7] ${color}[${level.toUpperCase()}]§r ${msg}`)
}

// ================= 安全睡眠 =================
function safeSleep(ms, step = 50) {
    const end = Time.time() + ms
    while (isEnabled() && Time.time() < end) {
        Time.sleep(Math.min(step, end - Time.time()))
    }
    return isEnabled()
}

// ================= 监听器注册区 =================
if (isEnabled()) {
    JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
        if (!isEnabled()) return
        if (!e.text) return
        // 聊天监听逻辑写这里
        log(`收到消息: ${e.text.getString()}`, "debug")
    }))
}

// ================= 主循环 =================
while (isEnabled()) {
    if (!World.isWorldLoaded()) { Client.waitTick(1); continue }
    if (!Player.getPlayer()) { Client.waitTick(1); continue }

    // 主逻辑写这里
    log("Hello World", "debug")

    if (!safeSleep(200)) break
}
```

## 逐段讲解

### 开关：GlobalVars.toggleBoolean

```javascript
GlobalVars.toggleBoolean(scriptName)
```

`GlobalVars` 是**所有脚本共享**的一块内存（详见[全局变量共享](globals.md)）。`toggleBoolean(name)` 把里面的一个布尔值取反并返回新值（签名：`toggleBoolean(name: string): boolean | null`）。

按键每按一次，就是**一次全新的脚本运行**。于是整个流程变成：

1. 第一次运行：键还不存在，取反后变成 `true`，本次运行进入主循环，成为"工作线程"。
2. 第二次运行：取反成 `false`，本次运行走关闭分支，几乎立刻结束；而上一次运行的主循环在下一次检查 `isEnabled()` 时发现开关关了，自己退出。

!!! note "关键理解"
    "关闭脚本"不是新一次运行把旧线程杀掉，而是它把共享开关拨到 `false`，旧线程**自己看见后退出**。所以旧线程里每一个循环、每一次长等待，都必须有机会看到这个开关。

### isEnabled 为什么是函数不是变量

```javascript
function isEnabled() {
    return GlobalVars.getBoolean(scriptName)
}
```

如果写成 `const enabled = GlobalVars.getBoolean(scriptName)`，取出来的只是运行那一刻的快照，之后再也感知不到变化。必须每次都去 `GlobalVars` 现查，开关才能"实时生效"。

### 关闭分支：disableScriptListeners

```javascript
JsMacros.disableScriptListeners()
```

关闭那次运行里调用它，会移除**所有由脚本注册的**监听器（包括 `on`、`once`、`waitForEvent` 创建的），把上一轮注册的监听器一并清掉。

!!! warning "它会波及其他脚本"
    按 d.ts 注释，`disableScriptListeners()` 移除的是"所有用户创建的监听器"，不只属于当前这个文件。如果你同时运行多个监听型脚本，更精确的做法是保存 `JsMacros.on(...)` 的返回值，用 `JsMacros.off(listener)` 只关自己的，见下文"监听器也要清理"。

### 日志分级

```javascript
log("准备就绪")            // info，平时可见
log("循环第 3 次", "debug") // debug，默认不显示
log("背包快满了", "warn")
log("找不到目标容器", "error")
```

比到处写 `Chat.log` 好在两点：

- **一键静音**：调试完把 `currentLogLevel` 改回 `LOG_LEVELS.info`，所有 `debug` 日志立刻消失，不用一行行删；
- **统一前缀和颜色**：多脚本同时运行时，一眼看出消息是谁发的。

!!! tip "让日志函数只做日志"
    有的模板会在 `log()` 里写 `if (!Player.getPlayer()) { Client.waitTick(1); return }`——本意是"没进世界时消息可能看不到"。但日志函数偷偷 `waitTick` 会阻塞调用它的线程，行为很难预料。推荐让 `log()` 只负责输出，世界/玩家判空放在主循环里做。

### safeSleep：为什么不能直接 Time.sleep 长睡

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
if (!safeSleep(5000)) break // 在循环里
if (!safeSleep(5000)) return // 在函数里
```

### 监听器注册区为什么包在 if (isEnabled()) 里

关闭那次运行也会从头到尾执行整个文件。如果注册语句裸露在外面，"关闭"的那次运行反而又注册了一份新监听器。所以：

- 注册语句包在 `if (isEnabled()) { ... }` 里，只有开启那轮才注册；
- 回调内部第一行再写 `if (!isEnabled()) return`，这样即使监听器没被及时移除，它也不会再干活。

### 主循环的判空

```javascript
if (!World.isWorldLoaded()) { Client.waitTick(1); continue }
if (!Player.getPlayer()) { Client.waitTick(1); continue }
```

脚本可能在主菜单、加载界面、切维度的瞬间运行，此时 `World.isWorldLoaded()` 为 `false`、`Player.getPlayer()` 为 `null`，直接调用玩家方法会报错。判空失败时用 `Client.waitTick(1)` 等下一个游戏 tick 再试，而不是空转烧 CPU。

## JsMacros 的多线程模型

可以先粗暴理解成这样（完整版见[脚本上下文](script_context.md)）：

| 行为 | 会发生什么 |
| --- | --- |
| 按键运行一个 fork 脚本 | 新开一个脚本执行上下文（线程） |
| 再按一次同一个脚本 | 又开一个新的上下文，**不会自动杀掉旧的** |
| `JsMacros.on(...)` | 注册一个监听器，事件来了以后再跑回调 |
| `JavaWrapper.methodToJavaAsync(...)` 再执行 | 额外跑一个异步任务 |
| `Time.sleep(10000)` | 当前线程睡 10 秒，期间不会检查你的开关 |

!!! warning "核心理解"
    `GlobalVars` 只是共享变量，不是线程杀手。`JsMacros.disableScriptListeners()` 只处理监听器，不会把正在 `Time.sleep()` 的主循环或异步任务从梦里拽出来。

## 监听器也要清理

只关自己注册的监听器，保存返回值再 `off`：

```javascript
const listener = JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    if (!isEnabled()) return
    if (e.text) Chat.log(e.text.getString())
}))
```

主循环退出时关闭它。推荐放在 `finally` 里——就算中间报错，也会尽力清理：

```javascript
try {
    while (isEnabled()) {
        if (!safeSleep(200)) break
    }
} finally {
    JsMacros.off(listener)
}
```

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
    `MethodWrapper` 在 Java 侧实现了 `Runnable` 等一系列函数式接口，所以可以 `.run()` 启动（d.ts 的类声明里只列出了 `accept` / `apply` / `test` 等，`run` 来自 Java 接口本身）。

!!! tip "记一句话"
    每一个 `while`、每一个长 `sleep`、每一个异步任务，都应该能回答：**脚本关闭时我怎么停？**

## 什么时候用 Client.waitTick？

`Client.waitTick(n)` 等**游戏 tick**，适合：

- 世界没加载、玩家为空时等待；
- 容器点击后等服务端同步；
- 想按"每秒 20 tick"的游戏节奏行动。

`safeSleep(ms)` 等**真实时间**，适合：

- 冷却时间、节流；
- 等聊天返回、等界面动画。

两者都不是"自动取消"，长等待仍然要围绕 `isEnabled()` 组织。

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

## 变体一：纯监听器型脚本

有的脚本不需要主循环，全部逻辑都由事件驱动（比如"收到某句话就回一句"）。模板可以简化成：

```javascript
const scriptName = "LISTENER_ONLY"
GlobalVars.toggleBoolean(scriptName)

function isEnabled() {
    return GlobalVars.getBoolean(scriptName)
}

if (isEnabled()) {
    Chat.log(`§7[§b${scriptName}§7] §a已开启`)

    JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
        if (!isEnabled()) return
        if (!e.text) return
        const msg = e.text.getString()
        if (msg.includes("你好")) {
            Chat.say("你好！")
        }
    }))
} else {
    Chat.log(`§7[§b${scriptName}§7] §c已关闭`)
    JsMacros.disableScriptListeners()
}
```

没有主循环，脚本文件执行完后，注册的监听器仍然存活，事件来了随时触发。关闭那次运行靠 `disableScriptListeners()` 收尾。

## 变体二：服务型脚本

常驻脚本更适合做成[服务](services.md)：随游戏启动自动运行，由服务管理器统一开关，不再需要 `GlobalVars` 开关。服务脚本里 `event` 是 `Service` 事件，专门提供了停止时的清理钩子：

```javascript
JsMacros.assertEvent(event, 'Service')

const draw = Hud.createDraw2D()
draw.setOnInit(JavaWrapper.methodToJava(() => {
    draw.addText("服务运行中", 10, 10, 0xFFFFFF, true)
}))
draw.register()

const listener = JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    if (!e.text) return
    // ...
}))

// 服务停止时：自动注销 draw，并清理本上下文的监听器
event.unregisterOnStop(true, draw)
```

`event.unregisterOnStop(offEvents, ...registrables)` 会在服务停止时按顺序执行清理；设置过它之后，服务即使跑到文件末尾也不会自动结束。细节见[服务](services.md)。

## 新手检查清单

- 每个循环条件里都有 `isEnabled()`。
- 每个长等待都用 `safeSleep()`，并检查返回值。
- 监听器注册包在 `if (isEnabled())` 里，回调第一行再判一次。
- 退出时 `JsMacros.off(listener)` 或 `disableScriptListeners()`。
- 每个异步任务都有退出条件。
- 操作世界前先 `World.isWorldLoaded()` 和 `Player.getPlayer()` 判空。
- 发现"关了再开出现双倍逻辑"，给脚本加 session。

下一步：把模板用起来，去[实战模式](practical_patterns.md)看常用代码套路。
