---
icon: lucide/route
---

# Baritone API

Baritone 是 Minecraft 的寻路 Mod。JsMacros 可以通过两种方式控制它：

| 方式 | 优点 | 缺点 |
| --- | --- | --- |
| 聊天命令桥接 | 简单，和手动输入 `#goto` 一样 | 字符串易写错，反馈不够结构化 |
| Java API 直调 | 可检查进程状态、设置目标、取消、改设置 | 需要 Baritone 类名和 Minecraft 映射匹配 |

!!! warning "使用边界"
    Baritone 可能明显改变玩家行为。多人服务器里先确认规则，别把寻路、跟随、自动攻击之类脚本丢到不允许的环境里。

## 安装和检查

先确认 Baritone 已经作为 Mod 加载：

```javascript
if (!Client.isModLoaded("baritone")) {
    Chat.log("没有加载 Baritone")
    return
}
```

官方 Baritone 文档里，聊天控制默认前缀是 `#`，例如 `#goto 1000 500`、`#stop`。如果命令没有响应，先检查 Baritone 是否安装、前缀控制是否开启，以及 `baritone/settings.txt` 里有没有关掉相关设置。

## 方式一：用聊天命令桥接

最简单：

```javascript
Chat.say("#goto 100 64 100")
```

停止：

```javascript
Chat.say("#stop")
```

跟随实体：

```javascript
Chat.say("#follow entity zombie skeleton")
```

设置项也可以走命令：

```javascript
Chat.say("#followRadius 6")
Chat.say("#allowSprint true")
```

!!! tip "为什么很多脚本仍用命令？"
    某些流程，比如按实体类型 follow，直接 Java API 要构造 Java predicate 或 Minecraft 原始实体类型；命令桥接反而更稳、更短。

## 方式二：Java API 直调

最小可用包装：

```javascript
const BaritoneAPI = Java.type("baritone.api.BaritoneAPI")
const GoalBlock = Java.type("baritone.api.pathing.goals.GoalBlock")
const SettingsUtil = Java.type("baritone.api.utils.SettingsUtil")

class BaritoneCtl {
    constructor() {
        this.baritone = BaritoneAPI.getProvider().getPrimaryBaritone()
    }

    cancel() {
        this.baritone.getPathingBehavior().cancelEverything()
    }

    goalProcess() {
        return this.baritone.getCustomGoalProcess()
    }

    gotoBlock(x, y, z) {
        this.goalProcess().setGoalAndPath(new GoalBlock(Math.floor(x), Math.floor(y), Math.floor(z)))
    }

    isPathing() {
        return this.goalProcess().isActive()
    }

    setting(name, value) {
        SettingsUtil.parseAndApply(BaritoneAPI.getSettings(), name, String(value))
    }
}

const baritone = new BaritoneCtl()
baritone.setting("followRadius", 6)
baritone.gotoBlock(100, 64, 100)
```

关键入口：

| API | 作用 |
| --- | --- |
| `BaritoneAPI.getProvider().getPrimaryBaritone()` | 获取主 Baritone 实例 |
| `BaritoneAPI.getSettings()` | 获取设置对象 |
| `baritone.getCustomGoalProcess()` | 自定义目标进程 |
| `goalProcess.setGoalAndPath(goal)` | 设置目标并开始寻路 |
| `goalProcess.isActive()` | 是否仍在执行 |
| `baritone.getPathingBehavior().cancelEverything()` | 取消所有寻路 |
| `SettingsUtil.parseAndApply(settings, name, value)` | 按字符串解析并应用设置 |

## 等待寻路完成

不要发完 `setGoalAndPath` 就假设到了。实际脚本要等进程结束、加超时、处理玩家离开世界。

```javascript
function waitPathDone(baritone, timeoutMs = 30000) {
    const start = Time.time()
    while (baritone.isPathing()) {
        if (!World.isWorldLoaded() || !Player.getPlayer()) {
            baritone.cancel()
            return false
        }

        if (Time.time() - start > timeoutMs) {
            baritone.cancel()
            Chat.log("Baritone 寻路超时，已取消")
            return false
        }

        Time.sleep(100)
    }

    return true
}

const b = new BaritoneCtl()
b.gotoBlock(100, 64, 100)
if (waitPathDone(b, 60000)) {
    Chat.log("到达目标附近")
}
```

## 卡住检测

自动钓鱼和自动拾取类脚本常用“位置变化太小”判断疑似卡住。

```javascript
function waitPathWithStuckHint(baritone, timeoutMs = 60000) {
    const start = Time.time()
    let lastPos = Player.getPlayer()?.getPos()
    let stuckTicks = 0

    while (baritone.isPathing()) {
        const player = Player.getPlayer()
        if (!player) {
            baritone.cancel()
            return false
        }

        const now = player.getPos()
        if (lastPos) {
            const dx = Math.abs(now.getX() - lastPos.getX())
            const dz = Math.abs(now.getZ() - lastPos.getZ())
            if (dx < 0.05 && dz < 0.05) {
                stuckTicks++
                if (stuckTicks > 50) {
                    Chat.actionbar("Baritone 疑似卡住")
                }
            } else {
                stuckTicks = 0
            }
        }

        if (Time.time() - start > timeoutMs) {
            baritone.cancel()
            return false
        }

        lastPos = now
        Time.sleep(100)
    }

    return true
}
```

## GoalNear

`GoalBlock(x, y, z)` 要求到一个具体方块。只需要到附近时，可以用 `GoalNear`。

```javascript
const GoalNear = Java.type("baritone.api.pathing.goals.GoalNear")

// 1.21.x Fabric intermediary 里 BlockPos 常见为 net.minecraft.class_2338。
// 如果 Java.type 失败，说明映射或版本不匹配，先换回 GoalBlock。
const BlockPos = Java.type("net.minecraft.class_2338")

const pos = new BlockPos(100, 64, 100)
const goal = new GoalNear(pos, 3)

const baritone = new BaritoneCtl()
baritone.goalProcess().setGoalAndPath(goal)
```

!!! warning "Minecraft 类名会变"
    `baritone.api.*` 相对稳定；`net.minecraft.class_2338` 这类 Minecraft intermediary 名更依赖版本和加载器。文档示例来自 1.21.x 常见环境，不保证所有版本都能直接跑。

## 强制输入

Baritone 的输入覆盖可以控制跳跃、潜行等状态。用完必须恢复。

```javascript
const Input = Java.type("baritone.api.utils.input.Input")
const baritone = BaritoneAPI.getProvider().getPrimaryBaritone()

baritone.getInputOverrideHandler().setInputForceState(Input.JUMP, true)
Time.sleep(1000)
baritone.getInputOverrideHandler().setInputForceState(Input.JUMP, false)
```

如果脚本可能被中途关闭，务必在停止逻辑里清理：

```javascript
try {
    baritone.getInputOverrideHandler().setInputForceState(Input.JUMP, true)
    Time.sleep(1000)
} finally {
    baritone.getInputOverrideHandler().setInputForceState(Input.JUMP, false)
}
```

## 服务脚本清理

服务脚本里可以用 `event.stopListener` 收尾：

```javascript
const b = new BaritoneCtl()
const Input = Java.type("baritone.api.utils.input.Input")

event.stopListener = JavaWrapper.methodToJava(() => {
    b.cancel()
    b.baritone.getInputOverrideHandler().setInputForceState(Input.JUMP, false)
})
```

普通按键脚本则建议用开关状态控制，并在关闭分支里取消：

```javascript
const enabled = GlobalVars.toggleBoolean("my_baritone_script")
const b = new BaritoneCtl()

if (!enabled) {
    b.cancel()
    Chat.log("脚本已关闭，Baritone 已取消")
    return
}
```

## 常见问题

| 现象 | 排查 |
| --- | --- |
| `Java.type("baritone.api.BaritoneAPI")` 报错 | Baritone 没加载，或装的是不带 API 的版本 |
| `#goto` 变成公开聊天 | 前缀/聊天控制设置不对，优先用 `#` 前缀 |
| `GoalNear` 的 `BlockPos` 找不到 | Minecraft 映射类名不匹配，先用 `GoalBlock` |
| 寻路永远 active | 加超时，必要时 `cancelEverything()` |
| 脚本停了还在跳 | 清理 `InputOverrideHandler` |
| 跟随实体 API 难写 | 优先用 `Chat.say("#follow entity ...")` |

## 推荐写法小结

- 用 `BaritoneCtl` 这类小包装集中管理 Baritone。
- 到点寻路优先 `GoalBlock`，附近寻路再考虑 `GoalNear`。
- 等待寻路时加 `timeout`、世界/玩家判空、卡住提示。
- 关闭脚本时同时取消寻路和清理强制输入。
- 对实体跟随这类命令语义强的功能，命令桥接常常比硬写 Java predicate 更稳。

## 参考链接

- [Baritone GitHub](https://github.com/cabaletta/baritone)
- [Baritone API Javadocs](https://baritone.leijurv.com/)

