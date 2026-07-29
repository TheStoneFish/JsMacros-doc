---
icon: lucide/map
---

# API 总览

JsMacros 会在每个脚本里注入一批全局对象，不需要 `import`，直接用就行。本页是一张"地图"：先列出全部全局命名空间，再按"我想做什么"给出任务导向的索引。

## 全局变量

除了库，还有三个直接注入的全局变量：

| 名称 | 类型 | 什么时候有用 |
| --- | --- | --- |
| `event` | `Events.BaseEvent` 的具体子类型 | 事件触发的脚本里读取触发信息（聊天内容、按键、槽位……），详见[事件系统](events.md) |
| `context` | `EventContainer` | 控制脚本生命周期、释放 joined 事件锁，详见[脚本上下文](script_context.md) |
| `file` | `java.io.File` | 当前脚本文件对应的 Java 文件对象 |

按键手动运行的脚本里，`event` 只是基础事件；在 `JsMacros.on` 的回调里，优先使用回调参数 `(e, context)` 而不是全局 `event`。

## 全局库地图

| 库 | 一句话职责 | 常用入口 | 文档页 |
| --- | --- | --- | --- |
| `Chat` | 聊天输出、发消息、标题、构造文本组件 | `log`、`say`、`actionbar`、`title`、`createTextBuilder` | [聊天与文本](chat.md) |
| `Client` | 客户端本身：tick 等待、主线程、连接服务器、剪贴板、Mod 列表、游戏设置 | `waitTick`、`runOnMainThread`、`connect`、`getGameOptions` | [客户端](client.md) |
| `FS` | 相对 `Macros` 目录的文件读写和遍历 | `open`、`exists`、`makeDir`、`list` | [文件系统](fs.md) |
| `GlobalVars` | 所有脚本共享的内存变量空间 | `putBoolean`、`getBoolean`、`toggleBoolean`、`getObject` | [全局变量共享](globals.md) |
| `Hud` | 2D/3D 叠加渲染、打开脚本 GUI、窗口与鼠标信息 | `createDraw2D`、`createDraw3D`、`createScreen` | [HUD 渲染](hud.md)、[脚本屏幕](screen.md) |
| `JavaUtils` | 创建 Java 集合、随机数、原始对象转 Helper | `createArrayList`、`createHashMap`、`getHelperFromRaw` | [时间与工具](time_utils.md) |
| `JavaWrapper` | 把 JS 函数包装成 Java 回调（写监听器必用） | `methodToJava`、`methodToJavaAsync`、`stop` | [脚本上下文](script_context.md) |
| `JsMacros` | 事件注册与等待、运行其他脚本、服务管理、自定义事件 | `on`、`once`、`off`、`waitForEvent`、`createCustomEvent` | [事件系统](events.md)、[服务](services.md) |
| `KeyBind` | 模拟按键、读取/修改按键绑定 | `pressKey`、`keyBind`、`getPressedKeys` | [键盘与鼠标输入](keybind.md) |
| `Player` | 本地玩家：状态、背包、交互、射线检测 | `getPlayer`、`openInventory`、`rayTraceBlock` | [玩家](player.md)、[背包](inventory.md)、[交互](interaction.md) |
| `PositionCommon` | 创建位置/向量对象 | `createPos`、`createVec` | [位置与向量](position.md) |
| `Reflection` | Java 反射、运行时编译类、加载 jar | `getClass`、`newInstance`、`createClassBuilder` | [Java 与反射](java_api.md) |
| `Request` | HTTP 请求和 WebSocket | `get`、`post`、`create`、`createWS` | [网络请求](network.md) |
| `Time` | 当前时间戳和阻塞睡眠 | `time`、`sleep` | [时间与工具](time_utils.md) |
| `Utils` | 杂项：哈希、Base64、猜测聊天发送者、非空断言 | `hashString`、`encode`、`decode`、`guessName` | [时间与工具](time_utils.md) |
| `World` | 世界：方块、实体、玩家列表、维度、光照、声音 | `getBlock`、`findBlocksMatching`、`getEntities`、`isWorldLoaded` | [世界与方块](world.md)、[方块](blocks.md)、[实体](entities.md) |


## 我想做 X，该看哪页？

| 我想…… | 去这页 |
| --- | --- |
| 在聊天栏输出调试信息 / 发送聊天和命令 | [聊天与文本](chat.md) |
| 监听聊天消息并做出反应 | [事件系统](events.md)（`RecvMessage`）|
| 查全部事件的名称/参数/方法 | [全部事件参考](events_reference.md) |
| 给高频事件加过滤条件、降低开销 | [事件系统](events.md) |
| 写一个能"按一次开、再按一次关"的脚本 | [脚本模板](script_template.md)、[全局变量共享](globals.md) |
| 自动点击容器、整理背包、丢物品 | [背包](inventory.md) |
| 自动挖矿、走路、转头、攻击 | [交互](interaction.md)、[玩家](player.md) |
| 找附近的方块 / 扫描区域 | [世界与方块](world.md)、[方块](blocks.md) |
| 找附近的实体 / 读实体属性 | [实体](entities.md)、[特化实体参考](entities_reference.md) |
| 在屏幕上画血条、坐标、ESP 框 | [HUD 渲染](hud.md) |
| 做一个带按钮的设置界面 | [脚本屏幕](screen.md) |
| 发 HTTP 请求、连 WebSocket、对接 Webhook | [网络请求](network.md) |
| 保存配置文件、写日志文件 | [文件系统](fs.md) |
| 让脚本随游戏启动自动运行 | [服务](services.md) |
| 定时执行 / 等待若干秒 | [时间与工具](time_utils.md)|
| 模拟按键、改键位 | [键盘与鼠标输入](keybind.md) |
| 读写游戏设置（视频、声音、视距……） | [游戏设置](options.md) |
| 监听 / 修改网络数据包 | [数据包](packets.md) |
| 调用任意 Java 类或其他 Mod 的 API | [Java 与反射](java_api.md)、[外部 API 总览](external_api.md) |
| 让 Baritone 帮我寻路 | [Baritone API](baritone_api.md) |
| 理解线程、joined、看门狗 | [脚本上下文](script_context.md) |
