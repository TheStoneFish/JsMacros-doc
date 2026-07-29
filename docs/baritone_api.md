---
icon: lucide/route
---

# Baritone API

Baritone 是 Minecraft 的寻路 Mod。JsMacros 可以通过两种方式控制它：

| 方式 | 优点 | 缺点 |
| --- | --- | --- |
| 聊天命令桥接 | 简单，和手动输入 `#goto` 一样 | 字符串易写错，反馈不够结构化 |
| Java API 直调 | 可检查进程状态、设置目标、取消、改设置 | 需要 Baritone 类名和 Minecraft 映射匹配 |

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
这是一个实例可以根据自己的需求增加。

```javascript
const BaritoneAPI = Java.type('baritone.api.BaritoneAPI')
const GoalBlock = Java.type('baritone.api.pathing.goals.GoalBlock')
const SettingsUtil = Java.type('baritone.api.utils.SettingsUtil')
const GoalNear = Java.type('baritone.api.pathing.goals.GoalNear')

class BaritoneUtils {
    constructor() {
        this.baritone = BaritoneAPI.getProvider().getPrimaryBaritone()
    }

    cancel() {
        Client.runOnMainThread(
            JavaWrapper.methodToJava(() => {
                this.baritone.getPathingBehavior().cancelEverything()
            }),
            true,
            1000
        )
    }

    goto(x, y, z) {
        Client.runOnMainThread(
            JavaWrapper.methodToJava(() => {
                this.baritone.getCustomGoalProcess().setGoalAndPath(new GoalBlock(x, y, z))
            }),
            true,
            1000
        )
    }

    followFilter(filter) {
        Client.runOnMainThread(
            JavaWrapper.methodToJava(() => {
                this.baritone.getFollowProcess().follow(filter)
            }),
            true,
            1000
        )
    }

    set(settingName, settingsValue) {
        Client.runOnMainThread(
            JavaWrapper.methodToJava(() => {
                const settings = BaritoneAPI.getSettings()
                SettingsUtil.parseAndApply(settings, settingName, settingsValue.toString())
            }),
            true,
            1000
        )
    }

    isMineActive() {
        let active = null
        Client.runOnMainThread(
            JavaWrapper.methodToJava(() => {
                active = this.baritone.getMineProcess().isActive()
            }),
            true,
            1000
        )
        return active
    }

    isFollowActive() {
        let active = null
        Client.runOnMainThread(
            JavaWrapper.methodToJava(() => {
                active = this.baritone.getFollowProcess().isActive()
            }),
            true,
            1000
        )
        return active
    }

    getMineProcess() {
        let process = null
        Client.runOnMainThread(
            JavaWrapper.methodToJava(() => {
                process = this.baritone.getMineProcess()
            }),
            true,
            1000
        )
        return process
    }

    mineByName(quantity, blocks) {
        Client.runOnMainThread(
            JavaWrapper.methodToJava(() => {
                this.baritone.getMineProcess().mineByName(quantity, blocks)
            }),
            true,
            1000
        )
    }

    isCustomGoalProcessActive() {
        let active = null
        Client.runOnMainThread(
            JavaWrapper.methodToJava(() => {
                active = this.baritone.getCustomGoalProcess().isActive()
            }),
            true,
            1000
        )
        return active
    }

    gotoNear(blockPos, range) {
        Client.runOnMainThread(
            JavaWrapper.methodToJava(() => {
                const goal = new GoalNear(blockPos, range)
                this.baritone.getCustomGoalProcess().setGoalAndPath(goal)
            }),
            true,
            1000
        )
    }
}
```
!!! tip "为什么建议在主线程调用 Baritone API？"

    Baritone 的大部分 API 并不是线程安全的。如果在不同线程中同时操作同一个 Process，可能会导致竞态条件（Race Condition），甚至抛出异常。
    例如，`FollowProcess` 正在主线程执行 `follow()` 时，如果脚本线程同时调用 `cancelEverything()`，内部状态可能正在修改，此时 `followFilter` 等成员可能被提前置为 `null`，最终导致 `NullPointerException` 或其他不可预期的问题。
    因此，**建议对同一个 Baritone 实例的所有操作都在同一线程完成**。最简单、也是最安全的做法，就是统一通过：
    ```javascript
    Client.runOnMainThread(
        JavaWrapper.methodToJava(() => {
            // 调用 Baritone API
        }),
        true,
        1000
    )
    ```
    这样可以保证 `goto()`、`follow()`、`mine()`、`cancelEverything()`、修改 Settings 等操作按顺序在主线程执行，避免多个线程同时修改 Baritone 内部状态。
    **简单来说：要么所有 Baritone API 都在脚本线程调用，要么都在主线程调用，不要混用。** 对于 JsMacros，推荐统一在主线程调用。
    
关键入口：

| API                                                 | 作用                     |
| --------------------------------------------------- | ---------------------- |
| `BaritoneAPI.getProvider().getPrimaryBaritone()`    | 获取主 Baritone 实例。       |
| `BaritoneAPI.getSettings()`                         | 获取全局设置对象。              |
| `baritone.getCustomGoalProcess()`                   | 获取自定义目标（寻路）进程。         |
| `baritone.getMineProcess()`                         | 获取挖矿进程。                |
| `baritone.getFollowProcess()`                       | 获取跟随进程。                |
| `baritone.getPathingBehavior()`                     | 获取寻路行为控制器，可用于取消当前任务。   |
| `goalProcess.setGoalAndPath(goal)`                  | 设置目标并立即开始寻路。           |
| `goalProcess.isActive()`                            | 判断当前 Goal 是否仍在执行。      |
| `mineProcess.mineByName(count, blocks)`             | 开始挖掘指定方块。              |
| `followProcess.follow(filter)`                      | 开始跟随符合条件的实体。           |
| `pathingBehavior.cancelEverything()`                | 取消所有寻路、挖矿、跟随等任务。       |
| `SettingsUtil.parseAndApply(settings, name, value)` | 按字符串解析并应用 Baritone 设置。 |


## 等待寻路完成

不要发完 `setGoalAndPath` 就假设到了。实际脚本要等进程结束。

```javascript
function waitPathDone(baritone, timeoutMs = 30000) {
    const start = Time.time()
    while (baritone.isCustomGoalProcessActive()) {
        if (Time.time() - start > timeoutMs) {
            baritone.cancel()
            Chat.log("Baritone 寻路超时，已取消")
            return false
        }
        Time.sleep(100)
    }
    return true
}

const b = new BaritoneUtils()
b.goto(100, 64, 100)
if (waitPathDone(b, 60000)) {
    Chat.log("到达目标附近")
}
```
## 自定义follow目标
很多情况下需要自定义追踪的目标 而不是通过类型来追踪, 比如追踪 名称叫 `将军` 的实体, 或者是追踪着火的怪物
``` java
    @Override
    public void follow(Predicate<Entity> filter) {
        this.filter = filter;
        this.into = false;
    }
```
默认过滤器只支持 LivingEntity 和 EntityType，无法满足复杂条件。此时可直接传入自定义的 Predicate<Entity>。
在 JsMacros 中，用 JavaWrapper.methodToJava 将 JS 函数包装为 Java 的 Predicate 即可：
JavaScript
```
function entityFilter(e) {
    return e.method_5477().getString().includes('Lv.')
}
const filter = JavaWrapper.methodToJava(entityFilter)
baritone.followFilter(filter)
```
methodToJava 包装后的函数已实现 Predicate 接口，可直接传给 follow 方法。

## 参考链接

- [Baritone GitHub](https://github.com/cabaletta/baritone)
- [Baritone API Javadocs](https://baritone.leijurv.com/)

