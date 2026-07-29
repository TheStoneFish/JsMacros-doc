---
icon: lucide/clock
---

# 时间与工具

`Time` 和 `Utils` 是 JsMacros 的杂项工具库：它们不直接操作游戏世界，但计时、睡眠、哈希、Base64、猜测聊天发送者这些"周边活"都靠它们。本页把两个库的方法**逐一列全**，并重点讲清新手最容易踩坑的 `Time.sleep()`。


## Time

### 方法全表

`Time` 只有两个方法，但都非常常用：

| 方法 | 返回值 | 说明 |
| --- | --- | --- |
| `time()` | `number` | 当前 Unix 时间戳，单位**毫秒** |
| `sleep(millis)` | `void` | 让**当前脚本线程**睡眠 `millis` 毫秒，可能抛出 `InterruptedException` |

### 给脚本测耗时

`Time.time()` 最直接的用途就是掐表：

```javascript
const start = Time.time()

// ...你的脚本逻辑...
Client.waitTick(100)

Chat.log(`耗时 ${Time.time() - start} ms`)
```

上面这个例子还顺便说明了 tick 和真实时间的关系：理想情况下 100 tick = 5000 ms，但客户端卡顿时实际耗时会明显大于 5000 ms——这正是 `waitTick` 和 `sleep` 的本质区别（见下文）。

### 时间戳格式化

`Time.time()` 返回的是一串毫秒数，想显示成"2026-07-26 21:30:05"有两种能直接跑的写法。

**写法一：纯 JS `Date`**（推荐，无需 Java 知识）：

```javascript
function formatTime(ms) {
    const d = new Date(ms)
    const pad = n => String(n).padStart(2, "0")
    return `${d.getFullYear()}-${pad(d.getMonth() + 1)}-${pad(d.getDate())} ` +
           `${pad(d.getHours())}:${pad(d.getMinutes())}:${pad(d.getSeconds())}`
}

Chat.log(formatTime(Time.time()))   // 例如 2026-07-26 21:30:05
```

**写法二：Java 的 `SimpleDateFormat`**（格式字符串更灵活，详见 [Java 与反射](java_api.md)）：

```javascript
const SimpleDateFormat = Packages.java.text.SimpleDateFormat
const sdf = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss")

Chat.log(sdf.format(new Packages.java.util.Date(Time.time())))
```

常见用途：给日志文件按日期命名（配合 [文件系统（FS）](fs.md)）：

```javascript
const sdf = new Packages.java.text.SimpleDateFormat("yyyy-MM-dd")
const fileName = `log-${sdf.format(new Packages.java.util.Date(Time.time()))}.txt`
FS.open(fileName).append(`[${new Date().toLocaleTimeString()}] 脚本启动\n`)
```

### sleep 深入：它到底阻塞了什么

JsMacros 里**每个脚本运行在自己的线程上**。`Time.sleep(1000)` 的意思是：让当前这条脚本线程原地睡 1 秒，期间这个脚本什么都不做——不响应开关、不检查条件、不处理事件。游戏本身照常运行，其他脚本也不受影响。

```javascript
Chat.log("开始")
Time.sleep(2000)   // 本脚本卡在这里 2 秒，游戏不卡
Chat.log("2 秒后")
```

!!! warning "绝对不要在 joined 回调 / 主线程任务里 sleep"
    以 **joined 模式**注册的事件监听、以及 `Client.runOnMainThread(...)` 里的代码，是在**游戏主线程等待**的情况下执行的。在这种上下文里调用 `Time.sleep()`，睡的就是游戏主线程——画面直接冻住；睡得超过看门狗时限（配置项 `watchdogMaxTime`），脚本会被直接杀掉。同理，`Client.waitTick()` 在这种上下文里会造成"主线程等脚本、脚本等 tick、tick 等主线程"的循环等待，把游戏卡死。详见 [客户端](client.md)。

### sleep 和 Client.waitTick 的区别

| | `Time.sleep(millis)` | `Client.waitTick(i)` |
| --- | --- | --- |
| 计量单位 | 毫秒（真实时间） | 客户端 tick（游戏时间，理想 1 tick = 50 ms） |
| 卡顿时 | 照睡不误，时长精确 | 1 tick 会被拉长，等待随游戏节奏对齐 |
| 适合场景 | 现实世界节奏：请求限流、日志间隔、防刷屏 | 游戏世界节奏：等动画、等背包刷新、逐 tick 循环 |
| 断开连接后 | 正常 | tick 可能不再推进，会一直等 |

一句话：**和游戏世界打交道用 `Client.waitTick()`，和现实世界打交道用 `Time.sleep()`**。`waitTick` 的完整用法见 [客户端](client.md)，这里不重复展开。

### 长睡眠请用"安全睡眠"

`Time.sleep(10000)` 期间线程完全睡死，**不会检查你的脚本开关**。你按快捷键"关闭"了脚本，旧线程还会把 10 秒睡满，醒来再执行一轮才退出——体验就是"我明明关了，它还在动"。正确姿势是把长睡切成小段、每段醒来检查一次开关：

```javascript
function safeSleep(totalMs, step) {
    step = step || 200
    const end = Time.time() + totalMs
    while (isEnabled() && Time.time() < end) {
        Time.sleep(Math.min(step, end - Time.time()))
    }
    return isEnabled()   // false = 睡眠期间脚本被关了
}
```

这段代码的完整来龙去脉（开关、监听器清理、session 防复活）见 [脚本模板](script_template.md)，写任何"会循环、会久等"的脚本前都建议先读它。

## Utils

`Utils` 提供哈希、Base64、空值断言和聊天名字猜测。下表**一个不漏**地列出了 2.1.0 的全部方法：

### 方法全表

| 方法 | 返回值 | 说明 |
| --- | --- | --- |
| `hashString(message)` | `string` | 用 SHA-256 哈希字符串，十六进制输出 |
| `hashString(message, algorithm)` | `string | null` | 指定算法哈希，十六进制输出；算法名无效时返回 `null` |
| `hashString(message, algorithm, base64)` | `string | null` | 同上，`base64` 为 `true` 时以 Base64 编码输出 |
| `encode(message)` | `string` | Base64 编码 |
| `decode(message)` | `string` | Base64 解码 |
| `requireNonNull(obj)` | `T` | `obj` 非空则原样返回，为 `null` 则抛 `NullPointerException` |
| `requireNonNull(obj, message)` | `T` | 同上，抛出的异常带自定义消息 |
| `guessName(text)` | `string | null` | 猜测一条聊天消息的发送者名字，猜不出返回 `null` |
| `guessNameAndRoles(text)` | `JavaList<string>` | 猜测发送者的名字及头衔/身份组，猜不出返回空列表 |

`algorithm` 可选值：`sha1`、`sha256`、`sha384`、`sha512`、`md2`、`md5`。

### 哈希与 Base64

```javascript
// 哈希：默认 SHA-256
Chat.log(Utils.hashString("hello"))                  // 2cf24dba5fb0a30e...（十六进制）
Chat.log(Utils.hashString("hello", "md5"))           // 5d41402abc4b2a76...
Chat.log(Utils.hashString("hello", "sha512", true))  // SHA-512，Base64 输出
Chat.log(Utils.hashString("hello", "不存在的算法"))    // null

// Base64 编解码
const encoded = Utils.encode("你好 JsMacros")
Chat.log(encoded)
Chat.log(Utils.decode(encoded))   // 你好 JsMacros
```

!!! tip "哈希能干什么"
    - 判断文件/字符串内容是否变化（记住上次的哈希，对比即可）；
    - 把长文本变成短的唯一标识，作为 `GlobalVars` 或缓存的键；
    - 注意 MD5/SHA1 只适合做校验，不要当密码存储用。

### requireNonNull：让空值尽早爆炸

很多 API 会返回 `null`（比如没猜到名字、没找到实体）。与其让 `null` 一路传下去、在很远的地方莫名报错，不如在源头断言：

```javascript
try {
    const name = Utils.requireNonNull(Utils.guessName("<无法解析>"), "没猜出发送者")
    Chat.log(`发送者: ${name}`)
} catch (e) {
    Chat.log("出错了: " + e)   // java.lang.NullPointerException: 没猜出发送者
}
```

### guessName：猜聊天消息的发送者

服务器聊天格式五花八门（`<Steve> 你好`、`[VIP] Steve: 你好`……），`guessName` 会尽力从中猜出玩家名，`guessNameAndRoles` 还会把头衔/身份组一起列出来：

```javascript
JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e, ctx) => {
    if (!e.text) return
    const raw = e.text.getString()

    const name = Utils.guessName(raw)
    if (name != null) Chat.log(`猜测发送者: ${name}`)

    const parts = Utils.guessNameAndRoles(raw)   // JavaList<string>
    if (!parts.isEmpty()) Chat.log(`名字 + 头衔: ${parts}`)
}))
```

!!! warning "只是'猜'"
    官方注释明确说了 *this is not guaranteed to work*。它对常见格式命中率不错，但对格式特殊的服务器，**用正则自己解析更可靠**。把它当省事的默认方案，不要当唯一方案。

## Utils.setClipboard

| 方法 | 返回值 | 说明 |
| --- | --- | --- |
| `Utils.getClipboard()` | `string` | 读取系统剪贴板内容 |
| `Utils.setClipboard(text)` | `void` | 把 `text` 写入系统剪贴板 |

### 实战：复制容器标题 + 鼠标下槽位 id

写商店/仓库类脚本时经常需要知道"这个箱子界面的标题是什么、我指着的是几号槽位"。把下面的脚本在 JsMacros 界面里绑定到一个按键，打开容器后按一下即可：

```javascript
// 打开箱子/商店等容器界面后触发
const inv = Player.openInventory()

const title = inv.getContainerTitle()
const slot = inv.getSlotUnderMouse()   // 鼠标不在任何槽位上时为无效值（通常是 -999）

Chat.log(`容器标题: ${title}`)
Chat.log(`鼠标下槽位 id: ${slot}`)

Client.setClipboard(title)
Chat.log("标题已复制到剪贴板，可直接粘贴进脚本里做匹配")
```

### 实战：复制当前坐标

```javascript
const pos = Player.getPlayer().getBlockPos()
const text = `${pos.getX()} ${pos.getY()} ${pos.getZ()}`

Client.setClipboard(text)
Chat.log(`已复制当前坐标: ${text}`)
```

读取剪贴板同样简单，比如把剪贴板里的坐标发到聊天框：

```javascript
Chat.say(`我在 ${Client.getClipboard()}`)
```

!!! note "打开文件 / 网址"
    `JsMacros.open(path)`（用系统默认程序打开文件）和 `JsMacros.openUrl(url)`（打开网址）在 2.1.0 中仍然可用，但已被标记为 **deprecated**。偶尔用用没问题，别在新脚本里重度依赖。

这两个库属于 Java 互操作的范畴，详细讲解在 [Java 与反射](java_api.md)，这里只留速查表。

## Java 集合与随机数：

```javascript
const list = JavaUtils.createArrayList()
list.add("a")
list.add("b")

const map = JavaUtils.createHashMap()
map.put("key", "value")

const random = JavaUtils.getRandom()
Chat.log(random.nextInt(100))
```

| 方法 | 作用 |
| --- | --- |
| `createArrayList()` / `createArrayList(array)` | Java ArrayList（可从 JS 数组创建） |
| `createHashMap()` | Java HashMap |
| `createHashSet()` | Java HashSet |
| `getRandom()` / `getRandom(seed)` | `SplittableRandom`，带种子时序列可复现 |
| `getHelperFromRaw(raw)` | 原始 Minecraft 对象转 helper，无对应 helper 时返回 `null` |
| `arrayToString(array)` | 数组转字符串 |
| `arrayDeepToString(array)` | 多维数组转字符串 |
