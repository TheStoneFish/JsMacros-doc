---
icon: lucide/message-square
---

# 聊天与文本

`Chat` 负责本地输出、发送聊天、打开聊天框、显示标题和构造 Minecraft 文本组件。

## 本地输出

```javascript
Chat.log("只显示在本地聊天栏")
Chat.logf("坐标: %.1f %.1f %.1f", 1.2, 64, -3.4)
Chat.logColor("&a绿色 &e黄色 &c红色")
```

| 方法 | 作用 |
| --- | --- |
| `log(message)` | 本地聊天栏输出 |
| `logf(message, ...args)` | 格式化输出 |
| `logColor(message)` | 支持颜色代码的输出 |
| `getLogger()` | 取得 SLF4J logger，适合调试 |

部分方法有 `await` 重载。需要等输出提交到主线程时传 `true`，普通脚本一般不需要。

## 发送聊天

```javascript
Chat.say("大家好")
Chat.say("/spawn")
```

`Chat.say` 会真的发给服务器。命令也走这个入口。

!!! warning "多人服提醒"
    `Chat.say()` 是真实发包，不是本地输出。测试脚本时先用 `Chat.log()`，确认无误再换成 `Chat.say()`。

## 打开聊天框

```javascript
Chat.open("/msg ")
```

`open(message)` 会打开聊天输入框，并预填文本。适合做快捷命令模板。

## 标题、ActionBar 和 Toast

```javascript
Chat.title("标题", "副标题", 10, 40, 10)
Chat.actionbar("显示在物品栏上方")
Chat.toast("JsMacros", "脚本已启动")
```

| 方法 | 参数 |
| --- | --- |
| `title(title, subtitle, fadeIn, remain, fadeOut)` | 标题、淡入、停留、淡出 tick |
| `actionbar(text)` | 显示 ActionBar |
| `actionbar(text, tinted)` | `tinted=true` 时带系统提示色 |
| `toast(title, desc)` | 右上角 Toast |
| `toast(title, desc, displayTimeMs)` | 指定显示毫秒数 |

## TextHelper

很多 API 返回 `TextHelper`，它不是普通字符串。

```javascript
const text = Chat.createTextHelperFromString("Hello")
Chat.log(text.getString())
Chat.log(text.getJson())
```

常用创建方法：

| 方法 | 用途 |
| --- | --- |
| `createTextHelperFromString(content)` | 普通文本 |
| `createTextHelperFromJSON(json)` | 从 Minecraft JSON 文本创建，失败返回 `null` |
| `createTextHelperFromTranslationKey(key, ...content)` | 从翻译键创建 |
| `createTextBuilder()` | 链式构造复杂文本 |

## 文本格式工具

```javascript
const raw = "&aHello &lJsMacros"
const section = Chat.ampersandToSectionSymbol(raw)
const plain = Chat.stripFormatting(section)
```

| 方法 | 作用 |
| --- | --- |
| `ampersandToSectionSymbol(string)` | `&a` 转 `§a` |
| `sectionSymbolToAmpersand(string)` | `§a` 转 `&a` |
| `stripFormatting(string)` | 去掉格式代码 |
| `getTextWidth(text)` | 计算文本渲染宽度 |

## 示例：聊天关键词提醒

```javascript
JsMacros.on("RecvMessage", JavaWrapper.methodToJava((e) => {
    const text = e.text.getString()
    if (text.includes("交易")) {
        Chat.actionbar("聊天里出现了交易关键词")
        Chat.toast("关键词", text)
    }
}))
```

