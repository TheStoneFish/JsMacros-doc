---
icon: lucide/cpu
---

# 脚本上下文与并发模型

每次运行脚本，JsMacros 都会为它创建一个独立的 Java 线程和一个"脚本上下文"。理解这套模型，才能解释很多新手常见的疑惑：为什么监听器会叠加、为什么 joined 回调卡住游戏、脚本跑完最后一行为什么还"活着"。本页讲清楚这些运行时机制。

## 三个内置全局变量

每个脚本一启动就自带三个全局变量：

| 变量 | 类型 | 说明 |
| --- | --- | --- |
| `context` | `EventContainer` | 当前脚本的事件容器，管理线程锁。注意：在 `JsMacros.on` 的回调里不要用这个全局的 `context`，要用回调的第二个参数 |
| `event` | `Events.BaseEvent` | 触发本脚本的事件。基类只有 `getEventName()`，用 `JsMacros.assertEvent` 断言成具体类型 |
| `file` | `java.io.File` | 当前脚本文件本身（Java 的 File 对象） |

```javascript
Chat.log(`事件名: ${event.getEventName()}`)
Chat.log(`脚本文件: ${file.getAbsolutePath()}`)
Chat.log(`是否持有事件锁: ${context.isLocked()}`)
```

## 线程模型：每次触发一个线程

JsMacros 的并发模型可以概括成三句话：

1. **每次脚本运行都是一个新线程。** 按键宏按两次、事件触发两次，就是两个互不相干的线程各跑一份脚本。
2. **脚本线程默认不阻塞游戏。** `Time.sleep(1000)`、`Client.waitTick()`、`JsMacros.waitForEvent(...)` 卡住的只是脚本自己的线程，游戏照常渲染。
3. **joined（加入线程）是例外。** 勾选 joined 的脚本/监听器会"加入"触发事件的线程，事件会等你的代码跑完（或释放锁）才继续——所以你才有机会 `cancel()` 掉 `SendMessage` 这类事件。

### joined 与看门狗
joined 脚本阻塞的是游戏线程本身，因此 JsMacros 用"看门狗"保护游戏：joined 脚本占住游戏线程太久（默认约 500 毫秒，可在 JsMacros 设置里调整）会被强制终止。`Client.runOnMainThread` 的 `watchdogMaxTime` 参数就是同一套机制：

```typescript
Client.runOnMainThread(runnable: MethodWrapper): void;
Client.runOnMainThread(runnable: MethodWrapper, watchdogMaxTime: long): void;  
Client.runOnMainThread(runnable: MethodWrapper, await: boolean, watchdogMaxTime: long): void;
```
典型用法——joined：

```javascript
// 绑定在 SendMessage 事件上、勾选了 joined 的脚本
JsMacros.assertEvent(event, 'SendMessage')

if (event.message.startsWith(".ping")) {
    // 1. 先取消事件
    event.cancel()         
    // 2. 释放锁, 游戏线程继续 
    context.releaseLock()  
     // 3. 之后再做耗时的事, 不卡游戏
    Time.sleep(1000)           
    Chat.log("pong!")
}
```
### 排队与优先级（JS/JEP 专用）

同一个脚本上下文里，JS 回调是排队执行的(事件循环)。`JavaWrapper` 提供了几个调度工具：

| 方法 | 说明 |
| --- | --- |
| `JavaWrapper.deferCurrentTask()` | 把当前任务放到队列末尾（小心别造成循环等待） |
| `JavaWrapper.deferCurrentTask(priorityAdjust)` | 同上，并调整优先级 |
| `JavaWrapper.getCurrentPriority()` | 获取当前任务优先级 |
| `JavaWrapper.methodToJavaAsync(priority, callback)` | 创建回调时直接指定队列优先级 |
| `JavaWrapper.stop()` | 关闭当前脚本上下文（见下文"脚本什么时候才算停止"） |

## 全局 `context` 能做什么

`context` 是 `EventContainer` 类型，本质是"脚本上下文 + 事件线程锁"的组合。全部方法：

| 方法 | 说明 |
| --- | --- |
| `isLocked()` | 是否仍持有锁（joined 事件在等你） |
| `releaseLock()` | 提前释放锁，让被 joined 阻塞的事件线程继续走 |
| `awaitLock(then)` | 等待锁被释放后执行 `then`（脚本里必须传 `MethodWrapper`）。**容易死锁，慎用** |
| `getCtx()` | 拿到底层的 `BaseScriptContext`（见下节） |
| `getLockThread()` / `setLockThread(thread)` | 查看/设置持锁线程，一般用不到 |


!!! note "回调里的 context 是另一个"
    `JsMacros.on("Xxx", JavaWrapper.methodToJava((e, ctx) => {...}))` 中回调的第二个参数 `ctx` 才是**这次事件触发**对应的容器；全局 `context` 是**注册监听的那个脚本**的容器。在回调里释放锁必须用 `ctx.releaseLock()`。

## 更底层：BaseScriptContext

`context.getCtx()` 返回 `BaseScriptContext`，也就是脚本上下文本体。常用成员：

| 成员 | 说明 |
| --- | --- |
| `startTime`（字段） | 脚本启动的时间戳 |
| `getTriggeringEvent()` | 触发本脚本的事件（等价于全局 `event`） |
| `getFile()` | 脚本文件（可能为 `null`，比如 `runScript` 直接跑字符串时） |
| `getContainedFolder()` | 脚本所在文件夹 |
| `getMainThread()` | 脚本主线程 |
| `getBoundThreads()` | 本上下文绑定的所有线程集合 |
| `getBoundEvents()` | 线程 → `EventContainer` 的映射（joined 时哪些线程绑了哪些事件） |
| `isMultiThreaded()` | 上下文是否允许多线程 |
| `isContextClosed()` | 上下文是否已关闭 |
| `closeContext()` | 强制关闭上下文（停止脚本、清理其监听器） |
| `eventListeners`（字段） | 本上下文注册的事件监听器表（WeakHashMap） |
| `shouldKeepAlive()` | 是否还有理由保持存活（有监听器等） |

还有一些更偏内部的方法：`bindThread` / `unbindThread` / `bindEvent` / `releaseBoundEventIfPresent` / `setMainThread` / `getSyncObject` / `clearSyncObject` / `wrapSleep`，日常脚本用不到。

```javascript
const ctx = context.getCtx()
Chat.log(`启动于: ${ctx.startTime}, 绑定线程数: ${ctx.getBoundThreads().size()}`)
```

## 脚本什么时候才算"停止"

**跑完最后一行不等于脚本结束。** 只要上下文里还挂着东西——`JsMacros.on` 注册的监听器、注册中的 Draw2D/Draw3D 等——上下文就保持存活（`shouldKeepAlive()`），监听器会一直触发。

!!! warning "监听器不会自动清理"
    这是最常见的坑：脚本里 `JsMacros.on("Tick", ...)` 之后，你改了代码再点一次运行——旧上下文没关，旧监听器还在，新上下文又注册了一个，于是回调触发两次。再点一次就是三次。
    监听器只在两种情况下消失：你手动 `JsMacros.off(listener)`，或它所属的**上下文被关闭**（GUI 里停止、`JavaWrapper.stop()`、`closeContext()`）。

结束一个脚本的几种方式：

| 方式 | 说明 |
| --- | --- |
| KeyBinding GUI 中停止 | JsMacros 界面里正在运行的脚本可以手动停止 |
| `JavaWrapper.stop()` | 脚本内部自杀式关闭自己的上下文 |
| `ctx.closeContext()` | 关闭任意拿得到的上下文 |
| `JsMacros.off(listener)` + 注销资源 | 不关上下文，只摘掉监听器；没有存活理由后上下文自然回收 |

!!! note "别一把梭全关"
    `getOpenContexts()` 返回的是**所有**未回收的上下文，包括正在运行的服务和当前脚本自己。遍历 `closeContext()` 之前先看清楚是谁。

## 用 runScript 从脚本启动另一个脚本

`JsMacros.runScript` 可以启动脚本文件，也可以直接运行一段代码字符串，返回新脚本的 `EventContainer`：

| 重载 | 说明 |
| --- | --- |
| `runScript(file)` | 运行脚本文件（路径相对宏文件夹） |
| `runScript(file, fakeEvent)` | 运行并把 `fakeEvent` 作为对方的 `event` 全局变量（推荐配合 `createCustomEvent` 传参） |
| `runScript(file, fakeEvent, callback)` | 同上，脚本结束时调用 `callback`，参数是抛出的异常（`Throwable`，正常结束为 `null`） |
| `runScript(language, script)` | 把字符串当脚本跑，`language` 如 `"js"` |
| `runScript(language, script, callback)` | 同上，带结束回调 |
| `runScript(language, script, file, callback)` | 指定伪文件路径（影响相对路径解析） |
| `runScript(language, script, file, event, callback)` | 最完整的重载 |

用自定义事件给子脚本传参数：

```javascript
// 父脚本
const args = JsMacros.createCustomEvent("myArgs")
args.putString("target", "Steve")
args.putInt("count", 3)

JsMacros.runScript("worker.js", args, JavaWrapper.methodToJava((err) => {
    if (err) Chat.log(`worker.js 出错: ${err}`)
    else Chat.log("worker.js 正常结束")
}))
```

```javascript
// worker.js
JsMacros.assertEvent(event, 'Custom')
const target = event.getString("target")
const count = event.getInt("count")
Chat.log(`收到参数: target=${target}, count=${count}`)
```

### wrapScriptRun：把脚本文件包成回调

`wrapScriptRun` / `wrapScriptRunAsync` 把一个脚本文件（或代码串）包装成 `MethodWrapper`，凡是要传回调的地方都能塞：

```javascript
// 每次 Tick 事件都同步运行 tick_handler.js
JsMacros.on("Tick", JsMacros.wrapScriptRun("tick_handler.js"))
```

重载：`wrapScriptRun(file)`、`wrapScriptRun(language, script)`、`wrapScriptRun(language, script, file)`，`wrapScriptRunAsync` 同款三个（异步版不阻塞调用方）。

## 事件脚本 vs 手动运行的区别

区别就在 `event` 全局变量里装的是什么：

| 启动方式 | `event` 的内容 |
| --- | --- |
| 事件触发（GUI 里绑定了事件） | 对应事件类型，带全部字段，如 `Events.RecvMessage` 的 `text` |
| 按键宏触发 | `Events.Key`（键名、按下/抬起等） |
| 服务启动 | `Events.Service`（见[服务脚本](services.md)） |
| GUI 手动点运行 / `runScript(file)` | 一个基础事件，基本只有 `getEventName()` 可用 |
| `runScript(file, fakeEvent)` | 你传进去的 `fakeEvent` |

想写"既能事件触发、也能手动测试"的脚本，先判断事件名再断言：

```javascript
if (event.getEventName() === "RecvMessage") {
    JsMacros.assertEvent(event, 'RecvMessage')
    Chat.log(`收到: ${event.text ? event.text.getString() : ""}`)
} else {
    Chat.log("手动运行, 使用测试数据")
}
```

### ScriptTrigger：GUI 里的一行宏

GUI 里每一条"事件/按键 → 脚本文件"的绑定，对应一个 `ScriptTrigger` 对象：

| 成员 | 说明 |
| --- | --- |
| `getTriggerType()` | 触发类型（见下表） |
| `getEvent()` | 绑定的事件名 |
| `getScriptFile()` / `getScriptPath()` | 脚本文件（字符串 / `Path`） |
| `getEnabled()` | 是否启用 |
| `joined`（字段） | 是否以 joined 方式运行 |

`ScriptTrigger$TriggerType` 枚举：

| 值 | 含义 |
| --- | --- |
| `KEY_RISING` | 按键按下时触发 |
| `KEY_FALLING` | 按键松开时触发 |
| `KEY_BOTH` | 按下和松开都触发 |
| `EVENT` | 事件触发 |

## 配置与档案入口

顺带一提，两个运行时相关的入口：

- `JsMacros.getProfile()` 返回 `BaseProfile`：`getCurrentProfileName()`、`renameCurrentProfile(name)`、`loadOrCreateProfile(name)`、`saveProfile()`、`triggerEvent(event)`（手动触发一个事件）、`logError(ex)` 等。
- `JsMacros.getConfig()` 返回 `ConfigManager`：`configFolder` / `macroFolder` / `configFile` 字段，`saveConfig()`、`loadConfig()`、`backupConfig()` 等。

```javascript
Chat.log(`当前档案: ${JsMacros.getProfile().getCurrentProfileName()}`)
Chat.log(`宏文件夹: ${JsMacros.getConfig().macroFolder.getAbsolutePath()}`)
```
!!! tip "相关页面"
    - 事件注册（`on` / `once` / `off` / `waitForEvent`）详见[事件系统](events.md)
    - 给事件加过滤条件详见[事件系统](events.md)
    - 常驻后台脚本详见[服务脚本](services.md)
