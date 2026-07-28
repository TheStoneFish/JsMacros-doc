---
icon: lucide/wrench
---

# 实战模式

这一页是"常用代码模式手册"：收集真实脚本里反复出现的写法，每个模式都按**问题 → 代码 → 讲解**展开。它不绑定某个玩法，只讲脚本怎么写得稳。配合[脚本模板](script_template.md)食用最佳。

## 开关脚本模式

**问题**：想让同一个按键"按一次开启、再按一次关闭"。

```javascript
const scriptName = "my_script"
const enabled = GlobalVars.toggleBoolean(scriptName)

Chat.log(enabled ? "脚本已开启" : "脚本已关闭")

if (!enabled) {
    JsMacros.disableScriptListeners()
}

while (GlobalVars.getBoolean(scriptName)) {
    // 主逻辑
    Client.waitTick(1)
}
```

**讲解**：`GlobalVars` 是所有脚本共享的内存，`toggleBoolean` 取反并返回新值。开启那次运行进入 `while` 循环干活；关闭那次运行取反成 `false`，清掉监听器后循环条件不成立，直接结束，而旧线程会在下一次检查时看到开关关了、自己退出。适合：自动钓鱼、HUD、后台扫描、辅助菜单。完整讲解（包括为什么只有 boolean 还不够）见[脚本模板](script_template.md)。

## 世界和玩家判空

**问题**：脚本崩在"还没进世界"或"切维度瞬间玩家为空"。

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

**讲解**：`Player.getPlayer()` 在主菜单、加载界面、切维度瞬间都可能返回 `null`。事件回调里也一样要判：

```javascript
JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    if (!World.isWorldLoaded() || !Player.getPlayer()) return
    if (!e.text) return
    Chat.log(e.text.getString())
}))
```

注意 `RecvMessage` 的 `text` 字段在 d.ts 里是 `TextHelper | null`，用之前也要判空。

## 等待条件成立：事件 vs 轮询

**问题**：做了一个动作（发命令、开箱子），需要等它的结果。

**方式一：waitForEvent（推荐，有事件可等时）**

```javascript
Chat.say("/bal")

const res = JsMacros.waitForEvent("RecvMessage", JavaWrapper.methodToJava(
    (e) => e.text != null && e.text.getString().includes("余额")
))

Chat.log(`拿到回复: ${res.event.text.getString()}`)
```

`JsMacros.waitForEvent(event, filter)` 会**阻塞当前线程**直到匹配的事件出现，返回值的 `.event` 就是那个事件对象。filter 回调返回 `true` 表示"就是它"。

!!! warning "先动手还是先等待？"
    上面这种"先 `say` 再 `waitForEvent`"在极端情况下可能漏掉响应（回复来得比等待注册还快）。对时序敏感的场景，用带 `runBeforeWaiting` 参数的重载：`JsMacros.waitForEvent(event, filter, runBeforeWaiting)`——它先注册好等待，再执行你的动作，绝不漏事件。

**方式二：轮询 + 超时（没有合适事件时）**

```javascript
function waitUntil(check, timeoutMs = 5000) {
    const start = Time.time()
    while (Time.time() - start < timeoutMs) {
        if (check()) return true
        Time.sleep(50)
    }
    return false
}

if (waitUntil(() => Player.openInventory().getContainerTitle().includes("商店"))) {
    Chat.log("商店打开了")
} else {
    Chat.log("等待超时")
}
```

**讲解**：任何等待外部状态的循环都必须有**超时**，否则服务器一卡，你的脚本就永远等下去。`waitForEvent` 精确、省 CPU；轮询通用、写法直白。能用事件用事件，等"没有对应事件的状态"（如某个方块变化后的世界状态）时才轮询。

## 安全睡眠

**问题**：`Time.sleep(10000)` 期间脚本感知不到开关已关闭。

```javascript
function safeSleep(ms, step = 50) {
    const end = Time.time() + ms
    while (GlobalVars.getBoolean("my_script") && Time.time() < end) {
        Time.sleep(Math.min(step, end - Time.time()))
    }
    return GlobalVars.getBoolean("my_script")
}

if (!safeSleep(5000)) { /* 脚本已关闭，收尾退出 */ }
```

**讲解**：把长睡拆成 50ms 的小段，每段之间检查开关，最多 50ms 内响应关闭。为什么这很重要、以及快速关开时旧线程复活的问题，见[脚本模板](script_template.md)。

## tick 对齐：Client.waitTick

**问题**：`Time.sleep` 等的是真实时间，和游戏节奏对不上。

```javascript
// 每秒（20 tick）刷新一次信息
while (GlobalVars.getBoolean("my_script")) {
    const p = Player.getPlayer()
    if (p) Chat.actionbar(`血量: ${p.getHealth().toFixed(1)}`)
    Client.waitTick(20)
}
```

**讲解**：`Client.waitTick(n)` 挂起脚本线程直到 n 个游戏 tick 过去。游戏里的一切（移动、红石、实体更新）都按 tick 推进，所以"等游戏状态变化"用 `waitTick`，"等现实时间"用 `sleep`。连续操作容器时插一两个 tick 能显著减少客户端/服务器不同步：

```javascript
const inv = Player.openInventory()
inv.click(13)
Client.waitTick(2)
inv.quick(20)
Client.waitTick(2)
```

## 防止重复触发：冷却时间戳

**问题**：事件在短时间内连续触发（比如同一句话刷屏），逻辑被执行了很多遍。

```javascript
let lastTriggered = 0
const COOLDOWN_MS = 3000

JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    if (!e.text) return
    if (!e.text.getString().includes("有人请求传送")) return

    const now = Time.time()
    if (now - lastTriggered < COOLDOWN_MS) return
    lastTriggered = now

    Chat.say("/tpaccept")
}))
```

**讲解**：记录上次触发的时间戳，间隔不够就直接 `return`。比"布尔锁 + 定时解锁"简单得多，也不需要额外线程。冷却值放到常量里，方便调整。

## 聊天消息驱动状态机

**问题**：一个流程要经历多个阶段，每个阶段等不同的服务器消息（正则提取数据）。

```javascript
// 阶段: idle -> waiting_confirm -> done
let state = "idle"

JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    if (!e.text) return
    const msg = e.text.getStringStripFormatting()

    if (state === "idle") {
        const m = msg.match(/^(\w+) 想要传送到你身边/)
        if (m) {
            Chat.log(`收到 ${m[1]} 的传送请求`)
            Chat.say("/tpaccept")
            state = "waiting_confirm"
        }
    } else if (state === "waiting_confirm") {
        if (msg.includes("传送成功")) {
            Chat.log("流程完成")
            state = "idle"
        }
    }
}))
```

**讲解**：

- 用一个 `state` 变量记录"现在进行到哪一步"，每条消息只和当前阶段的规则比对，不会串戏；
- 匹配服务器消息优先用 `getStringStripFormatting()`（去掉颜色代码后的纯文本），正则不用考虑 `§a` 之类的干扰；
- 正则捕获组（`m[1]`）用来提取玩家名、数字等数据；
- 状态多了以后，别忘了加"超时回到 idle"的兜底，否则某个阶段等不到消息就卡死在那一档。

## 容器操作套路

**问题**：自动操作箱子/商店界面，点快了、点错界面、点了个寂寞。

标准流程：**等界面打开 → 校验标题 → 找槽位 → 点击 → 等结果**。

```javascript
// 1. 等容器打开（先注册等待，再做触发动作也可以，见 waitForEvent 一节）
const opened = JsMacros.waitForEvent("OpenContainer")
const inv = opened.event.inventory

// 2. 校验标题，别在错误的界面上瞎点
if (!inv.getContainerTitle().includes("商店")) {
    Chat.log("§c打开的不是目标界面")
} else {
    // 3. 找槽位：按物品 id 找
    const slots = inv.findItem("minecraft:diamond")
    if (slots.length === 0) {
        Chat.log("§c界面里没有钻石")
    } else {
        // 4. 点击（quick = shift 点击）
        inv.quick(slots[0])

        // 5. 等服务器确认槽位变化，再做下一步
        JsMacros.waitForEvent("SlotUpdate")
        Client.waitTick(2)
        Chat.log("§a购买完成")
    }
}
```

没有合适事件时，用轮询版的"等标题"（原味老配方，配合超时）：

```javascript
function waitContainerTitle(name, timeoutMs = 3000) {
    const start = Time.time()
    while (Time.time() - start < timeoutMs) {
        if (!Player.getPlayer()) return false

        const inv = Player.openInventory()
        if (inv.getContainerTitle().includes(name)) return true

        Time.sleep(50)
    }
    return false
}

if (waitContainerTitle("确认购买")) {
    Player.openInventory().click(11)
}
```

**讲解**：

- `OpenContainer` 事件带 `inventory` 和 `screen` 字段，拿到的 `inventory` 就能直接操作；
- `findItem(id)` 返回槽位号数组（`JavaArray<number>`，有 `.length`、可下标访问）；
- `click(slot)` 是普通左键，`click(slot, 1)` 右键，`quick(slot)` 相当于 shift 点击；
- 每次点击之间 `Client.waitTick(1~2)`，等 `SlotUpdate` 确认后再进行依赖上一步结果的操作；
- 服务器菜单常见"点击后重开新界面"，此时旧的 `inv` 已失效，要重新等 `OpenContainer` 拿新的。槽位布局、`getMap()` 分区详见[背包](inventory.md)。

## 异常处理与 finally 清理

**问题**：脚本中途报错，HUD 残留在屏幕上、监听器成了孤儿。

```javascript
const draw = Hud.createDraw2D()
draw.setOnInit(JavaWrapper.methodToJava(() => {
    draw.addText("running", 10, 10, 0xFFFFFF, true)
}))
draw.register()

const listener = JsMacros.on("Tick", JavaWrapper.methodToJava(() => {
    // ...
}))

try {
    while (GlobalVars.getBoolean("my_script")) {
        Client.waitTick(20)
    }
} catch (err) {
    Chat.log(`§c脚本异常: ${err}`)
} finally {
    draw.unregister()
    JsMacros.off(listener)
}
```

**讲解**：`finally` 无论正常退出还是抛异常都会执行，把"注销 HUD、关监听器"这类清理放进去，脚本怎么死都不会留垃圾。服务脚本有更省心的写法——`event.unregisterOnStop(true, draw)` 让服务管理器在停止时自动清理（注意这是 `Service` 事件上的方法，见[服务](services.md)）。

## 后台工作线程

**问题**：有个慢活（大范围扫描、网络请求）不想卡住主循环。

```javascript
JavaWrapper.methodToJavaAsync(() => {
    while (GlobalVars.getBoolean("my_script")) {
        // 后台检查逻辑
        Time.sleep(200)
    }
}).run()
```

!!! warning "别忘了退出条件"
    异步循环必须有开关或停止条件，不然脚本关了它还在跑。每次长睡也应换成安全睡眠。

## 多脚本通信：GlobalVars + Custom 事件

**问题**：脚本 A 想把数据/信号传给脚本 B。

**共享状态用 GlobalVars**（B 随时来读）：

```javascript
// A 脚本
GlobalVars.putString("miner.status", "working")
GlobalVars.putInt("miner.count", 64)

// B 脚本
const status = GlobalVars.getString("miner.status")
```

**即时通知用自定义事件**（B 被动收到推送）：

```javascript
// B 脚本：先注册监听
JsMacros.on("MinerDone", JavaWrapper.methodToJava((e) => {
    Chat.log(`挖矿完成，共 ${e.getInt("count")} 个`)
}))
```

```javascript
// A 脚本：触发事件
const e = JsMacros.createCustomEvent("MinerDone")
e.putInt("count", 64)
e.trigger()
```

**讲解**：

- `GlobalVars` 是"拉"模型：适合开关、进度这类"谁需要谁来查"的状态；
- 自定义事件是"推"模型：`createCustomEvent(name)` 创建事件对象，`putXxx` 塞数据，`trigger()` 触发所有监听这个名字的回调；接收方用 `getInt` / `getString` / `getObject` 取数据；
- 首次使用可以调用 `e.registerEvent()` 把事件注册进 GUI 的事件列表，方便在界面里绑定脚本；
- TS 用户注意：`JsMacros.on` 的类型签名只认内置事件名，自定义事件名需要 `as any` 绕过类型检查（运行时完全正常）。

## 配置文件与日志文件

**问题**：想把设置和运行记录存到磁盘。

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

```javascript
function appendLog(message) {
    if (!FS.exists("logs")) FS.makeDir("logs")
    FS.open("logs/my_script.log").append(`[${new Date().toLocaleString()}] ${message}\n`)
}
```

!!! tip "append，不是 write"
    追加日志用 `append()`。`write()` 会覆盖整个文件。路径都是相对 `Macros` 目录的，详见[文件系统](fs.md)。

## 性能守则

**问题**：脚本一开，游戏帧数掉一半。

常见元凶和对策：

| 别这么干 | 改成这样 |
| --- | --- |
| 在 `Tick` 事件里每 tick 调 `World.findBlocksMatching(id, 8)` 大范围扫描 | 扫描放到主循环里低频跑（几秒一次），结果缓存复用；`chunkrange` 参数是**区块**半径，8 就是 17×17 个区块，能小则小 |
| 每 tick `World.getEntities()` 再自己过滤 | 用 `World.getEntities(distance)` 限制范围，或传过滤器重载 |
| 监听 `RecvPacket` / `Tick` 这类高频事件后在 JS 回调里做大量判断 | 用 [事件过滤器](event_filters.md)（`JsMacros.eventFilters()`）在 Java 侧先筛一遍 |
| 循环里不加任何等待，`while (true) {}` 空转 | 至少 `Client.waitTick(1)` 或 `Time.sleep(50)` |
| 每 tick `Chat.log` 刷屏 | 用 `Chat.actionbar` 显示实时数据，或加冷却 |
| joined 监听器里做耗时操作 | 先 `context.releaseLock()` 再干活（见[快速开始](quick_start.md)的看门狗一节） |

**讲解**：性能问题九成来自"高频 × 重操作"。要么降频（低频轮询 + 缓存），要么减重（缩小范围、Java 侧过滤），要么挪线程（fork/异步）。先用 `Time.time()` 前后打点量一下耗时，再优化。
