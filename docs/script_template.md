---
icon: lucide/file-code-2
---

# 脚本模板

很多 JsMacros 脚本不是“运行一次就结束”，而是会循环、监听事件、等容器打开、等 Baritone 寻路、等聊天返回。新手最容易踩的坑是：你以为脚本关了，其实旧线程还在 `Time.sleep()` 里面睡觉。

这一页用一个启停模板讲清楚它。

## 先看推荐模板

```javascript
const SCRIPT = "example_script"
const KEY_ENABLED = `${SCRIPT}.enabled`
const KEY_SESSION = `${SCRIPT}.session`

if (GlobalVars.getType(KEY_SESSION) !== "Int") {
    GlobalVars.putInt(KEY_SESSION, 0)
}

const nextEnabled = !GlobalVars.getBoolean(KEY_ENABLED)
GlobalVars.putBoolean(KEY_ENABLED, nextEnabled)
const session = GlobalVars.incrementAndGetInt(KEY_SESSION)

function isRunning() {
    return GlobalVars.getBoolean(KEY_ENABLED) === true
        && GlobalVars.getInt(KEY_SESSION) === session
}

function stopThisScript() {
    JsMacros.disableScriptListeners()
}

if (!nextEnabled) {
    Chat.log(`§7[§b${SCRIPT}§7] §cDisabled`)
    stopThisScript()
    return
}

Chat.log(`§7[§b${SCRIPT}§7] §aEnabled`)

const LOG_LEVELS = { debug: 0, info: 1, warn: 2, error: 3 }
let currentLogLevel = LOG_LEVELS.info

function log(message, level = "info") {
    const lvl = LOG_LEVELS[level] ?? LOG_LEVELS.info
    if (lvl < currentLogLevel) return

    let color = "§7"
    if (level === "debug") color = "§8"
    if (level === "warn") color = "§e"
    if (level === "error") color = "§c"

    Chat.log(`§7[§b${SCRIPT}§7] ${color}[${level.toUpperCase()}]§r ${message}`)
}

function safeSleep(ms, step = 50) {
    const end = Time.time() + ms
    while (isRunning() && Time.time() < end) {
        Time.sleep(Math.min(step, end - Time.time()))
    }
    return isRunning()
}

function ready() {
    return World.isWorldLoaded() && Player.getPlayer()
}

const listener = JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    if (!isRunning()) return
    if (!ready()) return
    // 这里写聊天监听逻辑
}))

try {
    while (isRunning()) {
        if (!ready()) {
            Client.waitTick(1)
            continue
        }

        log("Hello World", "debug")

        if (!safeSleep(200)) break
    }
} finally {
    JsMacros.off(listener)
    log("脚本线程已退出", "debug")
}
```

这个模板比最简单的 `GlobalVars.toggleBoolean(scriptName)` 多了一个 `session`。这不是花活，是防止老线程复活。

## 这版改进了什么？

你给的模板方向是对的：用 `GlobalVars.toggleBoolean()` 做开关，用 `safeSleep()` 拆分长等待，用 `disableScriptListeners()` 清监听器。这里把它改得更硬一点：

| 改动 | 为什么 |
| --- | --- |
| 把 `toggleBoolean` 的结果存成 `nextEnabled` | 关闭分支更直观，避免后面重复读状态读糊涂 |
| 新增 `KEY_SESSION` | 防止快速关开时旧线程醒来复活 |
| `safeSleep()` 返回 `boolean` | 调用方能立刻 `return` / `break` |
| 监听器保存到 `listener` 并在 `finally` 里 `off` | 比只靠关闭分支更稳，报错也会清 |
| `log()` 不在没玩家时 `waitTick` | 日志函数只负责日志，不偷偷阻塞当前线程 |

## 每一块在干什么？

| 代码 | 作用 |
| --- | --- |
| `KEY_ENABLED` | 这份脚本当前是否开启 |
| `KEY_SESSION` | 第几轮运行，用来区分新旧线程 |
| `nextEnabled` | 本次运行要切到开启还是关闭 |
| `session` | 本次运行拿到的运行代号 |
| `isRunning()` | 同时检查“开关是开着的”和“自己还是最新那轮” |
| `safeSleep()` | 分段睡眠，中途发现关闭就提前退出 |
| `ready()` | 世界和玩家判空 |
| `finally` | 无论正常退出还是报错，都清理监听器 |

## JsMacros 的多线程模型

可以先粗暴理解成这样：

| 行为 | 会发生什么 |
| --- | --- |
| 按键运行一个 fork 脚本 | 新开一个脚本执行上下文 |
| 再按一次同一个脚本 | 又开一个新的执行上下文，不会自动杀掉旧的 |
| `JsMacros.on(...)` | 注册一个监听器，事件来了以后再跑回调 |
| `JavaWrapper.methodToJavaAsync(...).run()` | 额外跑一个异步任务 |
| `Time.sleep(10000)` | 当前脚本线程睡 10 秒，期间它不会自动检查你的开关 |

所以，“关闭脚本”通常不是 JsMacros 帮你把所有线程暴力杀掉，而是你把共享状态改成关闭，然后让每个循环、监听器、异步任务自己看见这个状态并退出。

!!! warning "核心理解"
    `GlobalVars` 只是共享变量，不是线程杀手。`JsMacros.disableScriptListeners()` 只处理监听器，不会把正在 `Time.sleep()` 的主循环或异步循环从梦里拽出来。

## 为什么不能直接 Time.sleep？

看这个坏例子：

```javascript
while (GlobalVars.getBoolean("bad_script")) {
    Chat.log("开始等待")
    Time.sleep(10000)
    Chat.log("等待结束，继续执行")
}
```

如果你在 `Time.sleep(10000)` 期间再次运行脚本把开关关掉，旧线程不会立刻停。它还会睡满 10 秒。醒来以后虽然下一轮 `while` 会发现关闭，但 `Time.sleep` 后面那句 `Chat.log("等待结束，继续执行")` 仍然会跑。

更糟的是：

1. 旧线程开始睡 10 秒。
2. 你运行脚本关闭。
3. 你又马上运行脚本开启。
4. 旧线程醒来，发现 boolean 又是 true。
5. 旧线程和新线程一起跑，脚本变成双倍逻辑。

这就是为什么推荐模板里除了 boolean，还要有 `session`。

## safeSleep 的原理

`safeSleep(10000)` 不是真的一次睡 10 秒。它会切成很多小段：

```javascript
function safeSleep(ms, step = 50) {
    const end = Time.time() + ms
    while (isRunning() && Time.time() < end) {
        Time.sleep(Math.min(step, end - Time.time()))
    }
    return isRunning()
}
```

每睡一小段，就检查一次 `isRunning()`：

- 如果脚本仍然开启，而且 session 还是自己这一轮，就继续睡。
- 如果脚本被关闭，马上返回 `false`。
- 如果脚本被快速关开，session 变了，也返回 `false`，旧线程不会复活。

调用时要接返回值：

```javascript
if (!safeSleep(5000)) return
// 只有脚本仍然有效，才会执行这里
```

## boolean 开关为什么不够？

最简单模板通常长这样：

```javascript
const scriptName = "example"
GlobalVars.toggleBoolean(scriptName)

while (GlobalVars.getBoolean(scriptName)) {
    safeSleep(200)
}
```

这能解决“关闭后尽快退出”，但不能解决“快速关开后旧线程复活”。因为旧线程只看到了同一个 boolean，分不清“这是自己那轮开启”还是“用户后来新开的一轮”。

`session` 的作用就是给每轮运行贴一个号码：

```javascript
const session = GlobalVars.incrementAndGetInt(KEY_SESSION)

function isRunning() {
    return GlobalVars.getBoolean(KEY_ENABLED) === true
        && GlobalVars.getInt(KEY_SESSION) === session
}
```

新一轮脚本启动时，`KEY_SESSION` 会自增。旧线程手里的 `session` 过期，于是自动退出。

## 监听器也要清理

如果脚本注册了事件：

```javascript
const listener = JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    if (!isRunning()) return
    Chat.log(e.text.getString())
}))
```

退出时要关掉：

```javascript
JsMacros.off(listener)
```

或者关闭当前脚本注册的监听器：

```javascript
JsMacros.disableScriptListeners()
```

推荐在 `finally` 里清：

```javascript
try {
    while (isRunning()) {
        Client.waitTick(20)
    }
} finally {
    JsMacros.off(listener)
}
```

`finally` 的好处是：就算中间报错，也会尽力清理。

## 异步线程也要看 isRunning

如果你写了：

```javascript
JavaWrapper.methodToJavaAsync(() => {
    while (true) {
        Time.sleep(200)
    }
}).run()
```

这就是一个很危险的孤儿线程。它没有退出条件。

应该写成：

```javascript
JavaWrapper.methodToJavaAsync(() => {
    while (isRunning()) {
        // 后台检查逻辑
        if (!safeSleep(200)) return
    }
}).run()
```

!!! tip "记一句话"
    每一个 `while`，每一个长 `sleep`，每一个 async 任务，都应该能回答：脚本关闭时我怎么停？

## 什么时候用 Client.waitTick？

`Client.waitTick(1)` 适合等游戏下一 tick，常用于：

- 世界没加载时等待。
- 玩家对象为空时等待。
- 容器点击后等服务端同步。
- HUD 每秒更新。

`safeSleep(ms)` 适合等真实时间，常用于：

- 冷却时间。
- 等聊天返回。
- 等动画/界面打开。
- 长循环节流。

两者都不是“自动取消”。如果等待很长，仍然要检查 `isRunning()`。

## 新手检查清单

- 每个循环条件里都有 `isRunning()`。
- 每个长等待都用 `safeSleep()`，并检查返回值。
- 每个监听器退出时都 `off` 或 `disableScriptListeners()`。
- 每个异步任务都有退出条件。
- 每轮运行有 `session`，防止快速关开导致旧线程复活。
- 操作世界前先 `World.isWorldLoaded()` 和 `Player.getPlayer()` 判空。
