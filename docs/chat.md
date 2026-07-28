---
icon: lucide/message-square
---

# 聊天与文本

`Chat` 是 JsMacros 中最常用的全局对象之一，负责四件事：

| 能力 | 相关方法 |
| --- | --- |
| 输出到本地聊天栏（只有你自己能看到） | `log` / `logf` / `logColor` / `title` / `actionbar` / `toast` |
| 发送到服务器（所有人都能看到，命令也走这里） | `say` / `sayf` / `open` |
| 构造和解析 Minecraft 文本组件 | `TextHelper` / `TextBuilder` / `StyleHelper` |
| 操作聊天历史 | `getHistory()` → `ChatHistoryManager` |

自定义命令相关的 `getCommandManager()` 见 [自定义命令](commands.md)。

## 输出到本地

这些方法只在你自己的客户端显示，不会发出任何数据包，随便刷也不会被服务器踢。

```javascript
Chat.log("只显示在本地聊天栏")
Chat.log(Chat.createTextHelperFromString("也可以直接传 TextHelper"))
Chat.logf("坐标: %.1f %.1f %.1f", 1.2, 64.0, -3.4)
Chat.logColor("&a绿色 &e黄色 &c红色 &l&6加粗金色")
```

| 方法 | 说明 |
| --- | --- |
| `log(message)` | 输出到本地聊天栏，`message` 可以是字符串、`TextHelper` 等任意对象 |
| `log(message, await)` | `await=true` 时等消息真正提交到聊天栏后才继续执行 |
| `logf(message, ...args)` | 按 Java `String.format` 语法格式化后输出（`%s`、`%d`、`%.2f` 等） |
| `logf(message, await, ...args)` | 同上，带 `await` |
| `logColor(message)` | 自动把 `&` 颜色代码转换成 `§` 再输出，写彩色文本最方便 |
| `logColor(message, await)` | 同上，带 `await` |

!!! tip "await 参数是什么"
    聊天输出实际在主线程完成，脚本线程调用 `log` 后默认不等待。绝大多数脚本不需要关心；只有当你要求"这条消息必须已经显示，再执行下一步"（例如紧接着读聊天历史）时才传 `await=true`。

### 标题、ActionBar 和 Toast

```javascript
Chat.title("大标题", "副标题", 10, 40, 10)   // 淡入 10t、停留 40t、淡出 10t
Chat.actionbar("显示在物品栏上方")
Chat.actionbar("带底色的提示", true)
Chat.toast("JsMacros", "脚本已启动")
Chat.toast("JsMacros", "显示 5 秒", 5000)
```

| 方法 | 说明 |
| --- | --- |
| `title(title, subtitle, fadeIn, remain, fadeOut)` | 屏幕中央大标题，三个时间单位是 tick（20 tick = 1 秒） |
| `actionbar(text)` | 物品栏上方的小字提示 |
| `actionbar(text, tinted)` | `tinted=true` 时带系统提示的底色 |
| `toast(title, desc)` | 右上角弹出 Toast 通知 |
| `toast(title, desc, displayTimeMs)` | 指定显示时长（毫秒） |

## 发送到服务器

```javascript
Chat.say("大家好")          // 真实发送聊天消息
Chat.say("/spawn")          // 以 / 开头就是执行命令
Chat.sayf("我在 %d, %d, %d", 100, 64, -200)
```

| 方法 | 说明 |
| --- | --- |
| `say(message)` | 以玩家身份把消息发给服务器 |
| `say(message, await)` | `await=true` 时等消息真正发出后才继续 |
| `sayf(message, ...args)` | `String.format` 格式化后发送 |
| `sayf(message, await, ...args)` | 同上，带 `await` |

!!! warning "say 是真实发包"
    `Chat.say()` 和你亲手打字回车完全一样：消息以 `/` 开头就会执行命令，否则所有人都能看到。测试脚本时先用 `Chat.log()` 确认内容无误，再换成 `Chat.say()`。循环里高频 `say` 还可能触发服务器的刷屏检测。

### 打开聊天输入框

```javascript
Chat.open("/msg ")   // 打开聊天框并预填文本，等你补全后手动回车
```

| 方法 | 说明 |
| --- | --- |
| `open(message)` | 打开聊天输入框并预填 `message` |
| `open(message, await)` | 带 `await` 重载 |

官方 JSDoc 提示：可以配合 `JsMacros.waitForEvent("SendMessage")` 或 `JsMacros.once(...)` 等待玩家真正把消息发出去。

## § 颜色代码速查表

Minecraft 用 `§x` 控制颜色和格式。在脚本里推荐写 `&x` 然后用 `Chat.logColor` 或 `Chat.ampersandToSectionSymbol` 转换（源代码里直接打 `§` 容易出编码问题）。

| 代码 | 颜色 | 代码 | 颜色 |
| --- | --- | --- | --- |
| `§0` | 黑色 black | `§8` | 深灰 dark_gray |
| `§1` | 深蓝 dark_blue | `§9` | 蓝色 blue |
| `§2` | 深绿 dark_green | `§a` | 绿色 green |
| `§3` | 深青 dark_aqua | `§b` | 青色 aqua |
| `§4` | 深红 dark_red | `§c` | 红色 red |
| `§5` | 深紫 dark_purple | `§d` | 粉紫 light_purple |
| `§6` | 金色 gold | `§e` | 黄色 yellow |
| `§7` | 灰色 gray | `§f` | 白色 white |

| 代码 | 格式 |
| --- | --- |
| `§k` | 乱码滚动（obfuscated / magic） |
| `§l` | **加粗** |
| `§m` | ~~删除线~~ |
| `§n` | 下划线 |
| `§o` | *斜体* |
| `§r` | 重置全部格式 |

!!! tip
    颜色代码 `0`–`f` 正好对应十六进制 `0x0`–`0xf`，可以直接传给下文 `TextBuilder.withColor(int)`，例如金色是 `withColor(0x6)`、红色是 `withColor(0xc)`。

### 格式代码转换工具

```javascript
const raw = "&a你好 &l&6JsMacros"
const section = Chat.ampersandToSectionSymbol(raw)   // &a → §a
const back = Chat.sectionSymbolToAmpersand(section)  // §a → &a
const plain = Chat.stripFormatting(section)          // 去掉全部格式代码
const width = Chat.getTextWidth(plain)               // 渲染宽度（像素）
```

| 方法 | 说明 |
| --- | --- |
| `ampersandToSectionSymbol(string)` | `&` 转 `§`；1.9.0 起 `&&` 会被转义为字面的 `&` |
| `sectionSymbolToAmpersand(string)` | `§` 转 `&`；1.9.0 起会把 `&` 转义成 `&&` |
| `stripFormatting(string)` | 删除字符串中所有 `§x` 格式代码 |
| `getTextWidth(text)` | 返回文本的渲染宽度（像素），可用于对齐 |

## TextHelper：文本组件

很多 API（事件的 `text` 字段、物品名、告示牌内容……）返回的不是字符串而是 `TextHelper`——它是 Minecraft 文本组件的包装，除了文字还带颜色、悬停提示、点击事件等样式信息。

### 创建 TextHelper

| Chat 方法 | 说明 |
| --- | --- |
| `createTextHelperFromString(content)` | 从普通字符串创建 |
| `createTextHelperFromJSON(json)` | 从 Minecraft 原版 JSON 文本创建，解析失败返回 `null` |
| `createTextHelperFromTranslationKey(key, ...content)` | 从翻译键创建，如 `"item.minecraft.diamond"`，会随客户端语言变化 |
| `createTextBuilder()` | 返回 `TextBuilder`，链式构造复杂文本（见下节） |

### TextHelper 方法全表

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getString()` | string | 取纯文字内容（保留 `§` 内嵌格式码，如果有的话） |
| `getStringStripFormatting()` | string | 取文字并剥掉 `§` 格式码。有些老服务器用旧方式发彩色字，处理聊天时用这个最保险 |
| `getJson()` | string | 转成原版 JSON 文本表示，可存档后用 `createTextHelperFromJSON` 还原 |
| `withoutFormatting()` | TextHelper | 返回去掉全部样式的新 `TextHelper` |
| `visit(visitor)` | TextHelper | 遍历每个样式段，回调收 `(StyleHelper, string)` 两个参数，无返回值；用于逐段读取颜色/点击事件 |
| `getWidth()` | number | 渲染宽度（像素） |
| `toString()` | string | 字符串表示（1.0.8 起不再等价于 `getString()`，别拿它当文字用） |
| `toJson()` | string | 已弃用，改用 `getJson()` |
| `replaceFromString(content)` / `replaceFromJson(json)` | — | 已弃用，改用 `Chat.createTextHelperFromString/FromJSON` |

```javascript
JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    if (!e.text) return
    Chat.log("纯文本: " + e.text.getStringStripFormatting())
    Chat.log("JSON: " + e.text.getJson())
}))
```

### 用 visit 逐段读取样式

```javascript
const text = Chat.createTextBuilder()
    .append("红色").withColor(0xc)
    .append("加粗").withFormatting(false, true, false, false, false)
    .build()

text.visit(JavaWrapper.methodToJava((style, str) => {
    Chat.log(`段落 "${str}" 颜色索引=${style.getColorIndex()} 加粗=${style.bold()}`)
}))
```

## TextBuilder：链式构造复杂文本

`Chat.createTextBuilder()` 返回 `TextBuilder`，核心思路是：**`append` 开启一个新段落，之后的 `withXxx` 都作用在当前段落上**，最后 `build()` 得到 `TextHelper`。

| 方法 | 说明 |
| --- | --- |
| `append(text)` | 进入下一段并设置文字，可传字符串、`TextHelper` 或另一个 `TextBuilder` |
| `withColor(color)` | 用颜色索引上色，`0x0`–`0xf` 对应 `§0`–`§f`，如 `0x6` 金色、`0xc` 红色 |
| `withColor(r, g, b)` | 自定义 RGB 颜色，三个分量 0–255 |
| `withFormatting(underline, bold, italic, strikethrough, magic)` | 五个布尔值设置下划线/加粗/斜体/删除线/乱码 |
| `withFormatting(...formattings)` | 传入若干 `FormattingHelper` 设置格式 |
| `withShowTextHover(text)` | 鼠标悬停显示一段文本（参数须是 `TextHelper`） |
| `withShowItemHover(item)` | 悬停显示物品（参数是 `ItemStackHelper`） |
| `withShowEntityHover(entity)` | 悬停显示实体（参数是 `EntityHelper`） |
| `withClickEvent(action, value)` | 点击事件，`action` 见下表 |
| `withCustomClickEvent(callback)` | 点击时执行你自己的 JS 回调（`MethodWrapper`） |
| `withStyle(style)` | 直接套用一个 `StyleHelper` |
| `getWidth()` | 当前文本的渲染宽度 |
| `build()` | 构建为 `TextHelper` |

`withClickEvent` 支持的全部 action（来自官方 JSDoc）：

| action | 点击后 |
| --- | --- |
| `"open_url"` | 打开网址（value 为 URL） |
| `"open_file"` | 打开本地文件（value 为路径） |
| `"run_command"` | 立即执行命令 / 发送聊天（value 为完整命令） |
| `"suggest_command"` | 把 value 填进聊天输入框但不发送 |
| `"change_page"` | 翻页（用于成书界面，value 为页码） |
| `"copy_to_clipboard"` | 复制 value 到剪贴板 |

### 完整示例：可点击的聊天菜单

```javascript
const menu = Chat.createTextBuilder()
    .append("[菜单] ").withColor(0x6).withFormatting(false, true, false, false, false)
    .append("回城").withColor(0xa)
        .withShowTextHover(Chat.createTextHelperFromString("§a点击执行 /spawn"))
        .withClickEvent("run_command", "/spawn")
    .append(" | ").withColor(0x8)
    .append("私聊模板").withColor(0xb)
        .withShowTextHover(Chat.createTextHelperFromString("填入 /msg，自己补收件人"))
        .withClickEvent("suggest_command", "/msg ")
    .append(" | ").withColor(0x8)
    .append("复制坐标").withColor(0xe)
        .withClickEvent("copy_to_clipboard", "100 64 -200")
    .append(" | ").withColor(0x8)
    .append("Wiki").withColor(0xd)
        .withClickEvent("open_url", "https://jsmacros.wagyourtail.xyz/")
    .build()

Chat.log(menu)
```

### withCustomClickEvent：点击执行脚本

```javascript
const btn = Chat.createTextBuilder()
    .append("[点我算一下 2^10]").withColor(0xb)
    .withCustomClickEvent(JavaWrapper.methodToJava(() => {
        Chat.log("2^10 = " + Math.pow(2, 10))
    }))
    .build()
Chat.log(btn)
```

!!! warning
    `withCustomClickEvent` 的回调属于当前脚本上下文。普通脚本跑完就退出，回调会失效；这类交互建议放在常驻的服务（Service）里使用。

## StyleHelper：读取样式

`TextHelper.visit` 的回调、`TextBuilder.withStyle` 用到 `StyleHelper`，它是只读的样式信息。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `hasColor()` | boolean | 是否设置了颜色 |
| `getColorIndex()` | number | 颜色索引（0–15），没有颜色返回 `-1` |
| `getColorValue()` | number | RGB 颜色值，没有返回 `-1` |
| `getColorName()` | string \| null | 颜色名，如 `"gold"` |
| `getFormatting()` | FormattingHelper \| null | 对应的格式对象 |
| `hasCustomColor()` / `getCustomColor()` | boolean / number | 自定义（非 16 色）颜色 |
| `bold()` / `italic()` / `underlined()` / `strikethrough()` / `obfuscated()` | boolean | 五种格式开关 |
| `getClickAction()` | string \| null | 点击动作，可能是上表 6 种之一或 `"custom"` |
| `getClickValue()` | string \| null | 点击动作的 value |
| `getCustomClickValue()` | Runnable \| null | 自定义点击回调 |
| `getHoverAction()` / `getHoverValue()` | — | 悬停动作与内容 |
| `getInsertion()` | string | Shift+点击时插入聊天框的文本 |
| `getColor()` | number | 已弃用，改用 `getColorIndex()` |

`FormattingHelper`（`withFormatting(...)` 也接受它）的方法：`getName()`、`getCode()`（`§` 后面的那个字符）、`getColorIndex()`、`getColorValue()`、`isColor()`、`isModifier()`。

## 聊天历史操作

`Chat.getHistory()` 返回 `ChatHistoryManager`，可以读取、插入、删除聊天栏里已有的消息。

| 方法 | 说明 |
| --- | --- |
| `getRecvLine(index)` | 取第 `index` 条收到的消息（0 是最新的），返回 `ChatHudLineHelper` |
| `getRecvCount()` | 聊天历史中的消息条数 |
| `getRecvLines()` | 所有收到的消息列表 |
| `insertRecvText(index, line)` | 在 `index` 处插入一条 `TextHelper` |
| `insertRecvText(index, line, timeTicks)` | 指定创建时间 tick，官方提示：插入后最好调用 `refreshVisible()` |
| `insertRecvText(index, line, timeTicks, await)` | 带 `await` 重载 |
| `removeRecvText(index)` / `(index, await)` | 按下标删除 |
| `removeRecvTextMatching(text)` / `(text, await)` | 删除与 `TextHelper` 相同的消息 |
| `removeRecvTextMatchingFilter(filter)` / `(filter, await)` | 按回调过滤删除，回调收 `ChatHudLineHelper` 返回 `boolean` |
| `refreshVisible()` / `refreshVisible(await)` | 重建可见消息视图，改完历史后调用它才会刷新显示 |
| `clearRecv()` / `clearRecv(await)` | 清空收到的消息 |
| `getSent()` | 已发送消息历史（按 ↑ 翻的那个列表），返回的是直接引用，改它就是改历史 |
| `clearSent()` / `clearSent(await)` | 清空发送历史 |

`ChatHudLineHelper`（单条聊天记录）：

| 方法 | 说明 |
| --- | --- |
| `getText()` | 该行的 `TextHelper` |
| `getCreationTick()` | 该行创建时的 tick |
| `deleteById()` | 删除这一行 |

```javascript
// 把聊天栏里所有包含"广告"的历史消息清掉
const history = Chat.getHistory()
history.removeRecvTextMatchingFilter(JavaWrapper.methodToJava(
    (line) => line.getText().getStringStripFormatting().includes("广告")
))
history.refreshVisible()
```

## 日志（写到控制台 / latest.log）

| 方法 | 说明 |
| --- | --- |
| `getLogger()` | 返回 SLF4J Logger，输出到游戏日志（控制台和 `logs/latest.log`），不会出现在聊天栏 |
| `getLogger(name)` | 指定名字的 Logger，方便在日志里筛选自己的脚本 |

```javascript
const logger = Chat.getLogger("my-script")
logger.info("这行只出现在 latest.log 和控制台里")
logger.warn("警告级别")
```

!!! note
    2.1.0 的 `Chat` 没有直接写"聊天记录文件"的方法，需要持久化聊天内容请配合 [文件系统 FS](fs.md) 自己写文件。

## 实用示例

### 例 1：RecvMessage 过滤广告

事件详情见 [事件系统](events.md)。要拦截（取消）事件，注册时第二个参数传 `true`（joined 模式，回调和游戏同步执行）。

```javascript
// 建议保存为服务(Service)常驻运行
const AD_WORDS = ["低价代练", "点击领取", "www."]

JsMacros.on("RecvMessage", true, JavaWrapper.methodToJava((e) => {
    if (!e.text) return
    const msg = e.text.getStringStripFormatting()
    if (AD_WORDS.some((w) => msg.includes(w))) {
        e.cancel() // 这条消息不会显示
        Chat.getLogger("ad-filter").info("已拦截: " + msg)
    }
}))
```

### 例 2：SendMessage 本地命令拦截

把 `.` 开头的输入当成本地命令处理，不发给服务器。

```javascript
JsMacros.on("SendMessage", true, JavaWrapper.methodToJava((e) => {
    if (e.message == null || !e.message.startsWith(".")) return
    e.cancel() // 拦下来，服务器收不到

    const parts = e.message.substring(1).split(" ")
    switch (parts[0]) {
        case "pos": {
            const p = Player.getPlayer().getPos()
            Chat.log(`§b当前坐标: ${Math.floor(p.x)}, ${Math.floor(p.y)}, ${Math.floor(p.z)}`)
            break
        }
        case "clear":
            Chat.getHistory().clearRecv()
            break
        default:
            Chat.log("§c未知本地命令: ." + parts[0])
    }
}))
```

!!! tip
    想要带参数解析、Tab 自动补全的正式客户端命令，请使用 [自定义命令](commands.md) 里的 `CommandBuilder`。
