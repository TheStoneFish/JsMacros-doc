---
icon: lucide/database
---

# 全局变量共享（GlobalVars）

`GlobalVars` 是所有脚本共享的一块内存空间（d.ts 原注释：*"Global" variables for passing to other contexts*，即"用于传递给其他上下文的全局变量"）。它最适合做**开关、计数器、简单缓存和跨脚本状态**。

## 生命周期

!!! note "存在内存里，不落盘"
    - **跨脚本共享**：所有脚本（无论哪个文件、哪次运行）读写的都是同一个空间。
    - **游戏关闭即消失**：`GlobalVars` 只存在内存里，关闭客户端后全部清空；只要游戏进程还开着，换服务器、换存档都不会丢。
    - 需要长期保存的数据请用 [文件系统](fs.md) 写成 JSON 配置文件。

## 写入与读取

每种基础类型一对方法，读取**不存在的键返回 `null`**：

| 写入 | 读取 | 类型 |
| --- | --- | --- |
| `putInt(name, i)` | `getInt(name): number \| null` | 整数 |
| `putString(name, str)` | `getString(name): string \| null` | 字符串 |
| `putDouble(name, d)` | `getDouble(name): number \| null` | 浮点数 |
| `putBoolean(name, b)` | `getBoolean(name): boolean \| null` | 布尔值 |
| `putObject(name, o)` | `getObject(name): any` | 任意对象 |

```javascript
GlobalVars.putString("name", "Steve")
GlobalVars.putInt("count", 1)
GlobalVars.putBoolean("enabled", true)

Chat.log(GlobalVars.getString("name"))    // Steve
Chat.log(GlobalVars.getInt("count"))      // 1
Chat.log(GlobalVars.getBoolean("enabled"))// true
Chat.log(GlobalVars.getString("no_key"))  // null
```

!!! tip "复杂数据建议存 JSON 字符串"
    `putObject` 什么都能塞，但 JS 对象跨脚本（跨线程）访问可能受引擎限制。更稳妥的做法是 `putString(key, JSON.stringify(data))`，另一边 `JSON.parse(GlobalVars.getString(key))`。

## 开关脚本：toggleBoolean 模式

这是 `GlobalVars` 最经典的用法——**同一个按键，按一次开启脚本，再按一次停止**。

`toggleBoolean(name)` 会把布尔值取反并返回**新值**；键还不存在时，第一次调用相当于从"关"翻到"开"，得到 `true`。

通用模板（放在脚本最开头）：

```javascript
const scriptName = "MyScript"
GlobalVars.toggleBoolean(scriptName)
if (!GlobalVars.getBoolean(scriptName)) {
    JsMacros.disableScriptListeners()
    // 这里做清理工作，然后让脚本自然结束
}
```

### 原理：每按一次按键，就启动一个新的脚本实例

把脚本绑到按键上之后：

1. **第一次按键**：启动实例 A，`toggleBoolean` 把开关翻成 `true`，`if` 不命中，A 继续往下执行（注册监听器或进入循环）。
2. **第二次按键**：又启动一个全新的实例 B。B 的 `toggleBoolean` 把**共享的**开关翻成 `false`，`if` 命中——B 负责"关灯"后立刻退出。
3. 实例 A 并不知道 B 存在，它只能通过**不断重新读取 `GlobalVars.getBoolean(scriptName)`** 发现开关变成了 `false`，然后自己退出。这就是为什么循环里必须每圈都检查 `getBoolean`，而不能把值存进局部变量——局部变量不会被另一个实例改到，`GlobalVars` 才是两个实例之间唯一的"公告板"。

顺带一提，`getBoolean` 不存在时返回 `null`，而 `!null` 在 JS 里是 `true`，所以模板在任何初始状态下都能正确工作。

### 循环版完整示例

```javascript
const scriptName = "AutoActionbar"
GlobalVars.toggleBoolean(scriptName)

if (!GlobalVars.getBoolean(scriptName)) {
    Chat.log("§c脚本已停止")
    // 什么都不做，脚本运行到末尾自然退出
} else {
    Chat.log("§a脚本已启动，再按一次按键停止")
    while (GlobalVars.getBoolean(scriptName)) {   // 每圈都重新读开关
        Chat.actionbar("脚本运行中…")
        Client.waitTick(20)                        // 每秒一次，避免卡死游戏
    }
}
```

### 监听器版完整示例

事件驱动的脚本用 `JsMacros.disableScriptListeners()` 关闭——它会注销**当前脚本文件**注册的所有监听器（传事件名参数则只注销该事件的监听器）：

```javascript
const scriptName = "AttackCounter"
GlobalVars.toggleBoolean(scriptName)

if (!GlobalVars.getBoolean(scriptName)) {
    JsMacros.disableScriptListeners()
    Chat.log("§c攻击计数器已关闭")
} else {
    GlobalVars.putInt("attack_count", 0)
    JsMacros.on("AttackEntity", JavaWrapper.methodToJava(() => {
        const count = GlobalVars.incrementAndGetInt("attack_count")
        Chat.actionbar("攻击次数: " + count)
    }))
    Chat.log("§a攻击计数器已开启")
}
```

## 计数器方法

对整数键"读+改"一步完成，比 `getInt` 再 `putInt` 两步写法更简洁、更不容易出错：

| 方法 | 作用 |
| --- | --- |
| `getAndIncrementInt(name)` | 先取值，再加 1（返回加之前的值） |
| `incrementAndGetInt(name)` | 先加 1，再取值（返回加之后的值） |
| `getAndDecrementInt(name)` | 先取值，再减 1 |
| `decrementAndGetInt(name)` | 先减 1，再取值 |
| `toggleBoolean(name)` | 布尔值取反，返回新值 |

四个整数方法的返回值都是 `number | null`。

```javascript
GlobalVars.putInt("hits", 0)
Chat.log(GlobalVars.incrementAndGetInt("hits"))  // 1
Chat.log(GlobalVars.getAndIncrementInt("hits"))  // 1（返回旧值，存储变成 2）
Chat.log(GlobalVars.getInt("hits"))              // 2
```

## 其他方法

| 方法 | 返回值 | 作用 |
| --- | --- | --- |
| `getType(name)` | `string \| null` | 查看键的值类型，返回 `Int`、`String`、`Double`、`Boolean`、`Object` 或 `null` |
| `remove(key)` | `void` | 删除一个键 |
| `getRaw()` | `JavaMap<string, any>` | 取底层的 Java Map |

```javascript
GlobalVars.putBoolean("enabled", true)
Chat.log(GlobalVars.getType("enabled"))  // Boolean
GlobalVars.remove("enabled")

const raw = GlobalVars.getRaw()
Chat.log(raw.keySet())                   // 当前所有键
```

## 脚本间通信

`GlobalVars` 是**被动**共享：数据放在那里，另一个脚本得自己来读（轮询）。如果想"主动通知"另一个脚本，标准做法是 `GlobalVars` 搭配**自定义事件**——数据可以放进事件本身，也可以放进 `GlobalVars`：

发送方：

```javascript
const e = JsMacros.createCustomEvent("MyRelay")
e.putString("msg", "有人靠近了！")   // 数据直接随事件传递
e.trigger()
```

接收方（常驻脚本或[服务](services.md)）：

```javascript
JsMacros.on("MyRelay", JavaWrapper.methodToJava((event) => {
    Chat.log("收到消息: " + event.getString("msg"))
}))
```

自定义事件的用法详见[事件系统](events.md)。

## 命名建议

`GlobalVars` 是全体脚本共用的，键名冲突会互相覆盖。脚本多或多人协作时，键名加脚本前缀：

```javascript
const KEY_ENABLED = "my_script.enabled"
const KEY_SESSION = "my_script.session"
```

这样不容易和别的脚本撞名。
