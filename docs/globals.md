---
icon: lucide/database
---

# 全局变量

`GlobalVars` 是脚本间共享的一块内存。它最适合做开关、计数器、简单缓存和跨事件状态。

!!! note "生命周期"
    `GlobalVars` 不是写入磁盘的配置文件。重启客户端后不要假设数据还在。需要长期保存请用 [文件系统](fs.md)。

## 基础读写

```javascript
GlobalVars.putString("name", "Steve")
GlobalVars.putInt("count", 1)
GlobalVars.putBoolean("enabled", true)
GlobalVars.putObject("data", { ok: true })

Chat.log(GlobalVars.getString("name"))
Chat.log(GlobalVars.getInt("count"))
Chat.log(GlobalVars.getBoolean("enabled"))
```

| 写入 | 读取 |
| --- | --- |
| `putInt(name, i)` | `getInt(name)` |
| `putString(name, str)` | `getString(name)` |
| `putDouble(name, d)` | `getDouble(name)` |
| `putBoolean(name, b)` | `getBoolean(name)` |
| `putObject(name, o)` | `getObject(name)` |

读取不存在的键通常返回 `null`。

## 开关脚本模板

```javascript
const scriptName = "example_toggle"
const enabled = GlobalVars.toggleBoolean(scriptName)
Chat.log(enabled ? "脚本已开启" : "脚本已关闭")

while (GlobalVars.getBoolean(scriptName)) {
    Chat.actionbar("脚本运行中")
    Client.waitTick(20)
}
```

这是很多常驻小脚本最常用的结构：再次运行同一个脚本时，`toggleBoolean` 会把状态翻转。

## 计数器

```javascript
GlobalVars.putInt("hits", 0)

JsMacros.on("AttackEntity", JavaWrapper.methodToJava(() => {
    const count = GlobalVars.incrementAndGetInt("hits")
    Chat.actionbar(`攻击次数: ${count}`)
}))
```

可用方法：

| 方法 | 作用 |
| --- | --- |
| `getAndIncrementInt(name)` | 先取值，再加 1 |
| `incrementAndGetInt(name)` | 先加 1，再取值 |
| `getAndDecrementInt(name)` | 先取值，再减 1 |
| `decrementAndGetInt(name)` | 先减 1，再取值 |
| `toggleBoolean(name)` | 布尔值取反 |

## 删除和原始 Map

```javascript
GlobalVars.remove("enabled")

const raw = GlobalVars.getRaw()
Chat.log(raw.keySet())
```

`getType(name)` 可以查看值类型，返回 `Int`、`String`、`Double`、`Boolean`、`Object` 或 `null`。

## 命名建议

多人协作或脚本很多时，键名加前缀：

```javascript
const KEY_ENABLED = "my_script.enabled"
const KEY_SESSION = "my_script.session"
```

这样不容易和别的脚本撞名。

