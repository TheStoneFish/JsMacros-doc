---
icon: lucide/map
---

# API 总览

JsMacros 会在脚本里注入一批全局对象。平时写脚本时最常碰到的是 `Chat`、`Player`、`World`、`Hud`、`JsMacros`，其他库负责文件、网络、Java 互操作和工具函数。

!!! note "API 真相源"
    本页根据 `JsMacros-2.1.0.d.ts` 整理。签名里出现的 `JavaList`、`JavaMap`、`TextHelper`、`ItemStackHelper` 等类型不是普通 JS 对象，使用前建议先看 [常用类型](types.md)。

## 全局变量

| 名称 | 类型 | 什么时候有用 |
| --- | --- | --- |
| `event` | `Events.BaseEvent` 的具体子类型 | 事件脚本里读取触发信息，比如聊天内容、按键、槽位 |
| `context` | `EventContainer` | 控制事件脚本生命周期、释放 joined 事件锁 |
| `file` | `java.io.File` | 当前脚本文件对应的 Java 文件对象 |

在普通手动运行脚本里，`event` 通常只是基础事件；在事件监听回调里，应优先使用回调参数 `(e, context)`。

## 全局库速查

| 库 | 主要用途 | 常用入口 |
| --- | --- | --- |
| `Chat` | 聊天输出、发消息、标题、文本对象、命令构建 | `log`、`say`、`actionbar`、`createTextBuilder` |
| `Client` | 客户端状态、主线程、连接、剪贴板、Mod 列表 | `waitTick`、`runOnMainThread`、`connect`、`getGameOptions` |
| `FS` | 相对 jsMacros 目录的文件读写和遍历 | `open`、`list`、`exists`、`makeDir` |
| `GlobalVars` | 跨脚本共享内存状态 | `putBoolean`、`getObject`、`toggleBoolean` |
| `Hud` | 打开脚本 GUI、2D/3D 叠加渲染、鼠标窗口信息 | `createDraw2D`、`registerDraw2D`、`createScreen` |
| `JavaUtils` | 创建 Java 集合、随机数、原始对象 helper 包装 | `createArrayList`、`createHashMap` |
| `JavaWrapper` | 把 JS 函数包装成 Java 回调 | `methodToJava`、`methodToJavaAsync` |
| `JsMacros` | 运行脚本、注册事件、管理监听器和服务 | `on`、`once`、`runScript`、`waitForEvent` |
| `KeyBind` | 模拟按键、读取/修改原版按键绑定 | `pressKey`、`keyBind`、`getPressedKeys` |
| `Player` | 本地玩家、交互管理器、射线检测、移动预测 | `getPlayer`、`openInventory`、`rayTraceBlock` |
| `Reflection` | Java 反射、运行时编译、代理类、加载 jar | `getClass`、`getDeclaredMethod`、`invokeMethod` |
| `Request` | HTTP 和 WebSocket | `get`、`post`、`create`、`createWS` |
| `Time` | 当前时间和阻塞睡眠 | `time`、`sleep` |
| `Utils` | 哈希、Base64、空值断言、文本猜名 | `hashString`、`encode`、`decode` |
| `World` | 世界、方块、实体、时间、维度、声音 | `getBlock`、`findBlocksMatching`、`getEntities` |

## 一个脚本通常怎么组织？

短脚本可以直接顺序执行：

```javascript
const player = Player.getPlayer()
if (player) {
    Chat.log(`你在 ${player.getBlockPos()}`)
}
```

长时间运行的脚本要让出 Tick，避免把客户端卡住：

```javascript
while (true) {
    const player = Player.getPlayer()
    if (player) Chat.actionbar(`血量: ${player.getHealth()}`)
    Client.waitTick(20)
}
```

事件脚本用 `JsMacros.on` 注册：

```javascript
JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    const text = e.text.getString()
    if (text.includes("欢迎")) {
        Chat.log("检测到欢迎消息")
    }
}))
```

## 返回值的常见坑

| 现象 | 解释 | 写法 |
| --- | --- | --- |
| `Player.getPlayer()` 返回 `null` | 不在世界里，或客户端还没加载玩家 | `const p = Player.getPlayer(); if (!p) return` |
| `World.getEntities()` 返回 `null` | 世界未加载 | 先用 `World.isWorldLoaded()` 判断 |
| `TextHelper` 不能直接当字符串处理 | 它包装了 Minecraft 文本组件 | 用 `.getString()` 或 `.getStringStripFormatting()` |
| `JavaList` 不是 JS 数组 | 这是 Java 集合 | 可用 `.size()`、`.get(i)`，部分环境也能 `for...of` |
| `joined` 事件卡死 | joined 会阻塞主线程 | 修改事件后尽快 `context.releaseLock()` |

## 页面导读

- 想知道 `JsMacros.on` 怎么写，看 [事件系统](events.md)。
- 想看 `Inventory`、槽位、物品栈，看 [背包](inventory.md)。
- 想查方块、实体、维度和世界扫描，看 [世界与方块](world.md) 和 [实体](entities.md)。
- 想做屏幕显示、ESP、路径线，看 [HUD 渲染](hud.md)。
- 想访问 Java 类、反射、编译 Java，看 [Java 与反射](java_api.md)。

