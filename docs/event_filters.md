---
icon: lucide/filter
---

# 事件过滤器

`JsMacros.eventFilters()` 是 2.1.0 里很值得用的入口。它能在事件进入回调前先过滤，减少高频事件脚本里的判断和开销。

## 获取过滤器工厂

```javascript
const filters = JsMacros.eventFilters()
```

常用方法：

| 方法 | 作用 |
| --- | --- |
| `modulus(quotient)` | 每 `quotient` 次事件放行一次 |
| `limited(limit)` | 只放行前 `limit` 次 |
| `invert(base)` | 反转过滤结果 |
| `composed(initial)` | 组合过滤器 |
| `compile(code)` | 编译 Java 过滤代码 |
| `compile(event, code)` | 针对事件类型编译 |
| `getGlobalsForCompiled()` | 编译过滤器使用的全局数据 |

## 每 n 次触发一次

```javascript
const every20Ticks = JsMacros.eventFilters().modulus(20)

JsMacros.on("Tick", every20Ticks, JavaWrapper.methodToJava(() => {
    Chat.actionbar(`time=${World.getTimeOfDay()}`)
}))
```

这比在每个 Tick 回调里自己写计数器更清爽。

## 只触发前几次

```javascript
const firstThree = JsMacros.eventFilters().limited(3)

JsMacros.on("RecvMessage", firstThree, JavaWrapper.methodToJava((e) => {
    Chat.log(`前三条消息之一: ${e.text.getString()}`)
}))
```

## 和 joined 一起使用

`JsMacros.on` 的过滤器重载有两种：

```javascript
JsMacros.on("Tick", filter, callback)
JsMacros.on("SendMessage", filter, true, callback)
```

!!! warning "参数顺序"
    有过滤器时，`joined` 在过滤器后面：`on(event, filter, joined, callback)`。不要写成 `on(event, joined, filter, callback)`。

## compile 过滤器

`compile` 接收的是 Java 代码体，不是 JS 回调。它适合极高频事件或包事件，但调试成本更高。

```javascript
const filter = JsMacros.eventFilters().compile("Tick", "return true;")
JsMacros.on("Tick", filter, JavaWrapper.methodToJava(() => {
    Chat.log("tick")
}))
```

!!! tip "先用普通 JS"
    大多数脚本先用 `modulus`、`limited` 或普通回调判断就够了。只有性能真的成问题，再考虑 `compile`。

