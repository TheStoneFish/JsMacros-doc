---
icon: lucide/filter
---

# 事件过滤器

监听高频事件（`Tick` 每秒 20 次、`RecvPacket` 每秒可达上百次）时，往往只关心其中一小部分。过滤这件事有两条路：在 JS 回调里自己 `if`，或者用 2.1.0 新增的 `JsMacros.eventFilters()` 造一个 **Java 端过滤器**，在事件进你的脚本之前就把不相干的挡掉。本页把两条路讲清楚，并逐个介绍工厂的全部方法。

## 两种过滤方式

| | JS 回调谓词 | `eventFilters()` 工厂过滤器 |
| --- | --- | --- |
| 本质 | `JavaWrapper.methodToJava(e => 布尔值)` 包装的 `MethodWrapper` | Java 侧的 `EventFilter` 对象 |
| 执行位置 | 每个事件都要进你的 JS 上下文跑一次 | 事件线程上以 Java 速度执行，不进 JS |
| 开销 | 高频事件下可观 | 极低 |
| 灵活性 | 任意 JS 逻辑 | 固定几种（降频、限次、取反、组合）+ `compile` 的 Java 代码 |
| 用在哪 | `waitForEvent` 的 `filter` 参数 | `on` 的 `filter` 参数 |

!!! warning "两种过滤器不能互换"
    对照 d.ts 签名：`JsMacros.on(event, filter, ...)` 的 `filter` 是 `EventFilter`（只能来自 `eventFilters()` 工厂）；`JsMacros.waitForEvent(event, filter, ...)` 的 `filter` 是返回布尔值的 `MethodWrapper`（只能是 `JavaWrapper.methodToJava(...)` 包装的 JS 谓词）。传反了不会按你想的方式工作。

## JS 回调谓词：waitForEvent 的 filter

`waitForEvent` 会阻塞脚本线程直到"等到一个让谓词返回 `true` 的事件"，非常适合顺序流程。实战——等一条特定聊天消息：

```javascript
Chat.log("等待队伍邀请中...")

const result = JsMacros.waitForEvent(
    "RecvMessage",
    JavaWrapper.methodToJava((e) => e.text != null && e.text.getString().includes("邀请你加入队伍"))
)

Chat.log(`等到了: ${result.event.text.getString()}`)
Chat.say("/party accept")
```

谓词返回 `false` 的事件会被跳过继续等；`waitForEvent` 的重载、`join` 参数和返回值结构详见[事件系统](events.md)。

## Java 端过滤器：eventFilters() 工厂

```javascript
const f = JsMacros.eventFilters() // 返回 EventFilters 工厂
```

工厂全部方法（对照 d.ts，均为 2.1.0 新增）：

| 方法 | 返回 | 作用 |
| --- | --- | --- |
| `modulus(quotient)` | `FilterModulus` | 每 `quotient` 个事件放行一次 |
| `limited(limit)` | `FilterLimited` | 只放行前 `limit` 次 |
| `invert(base)` | `EventFilter` | 反转 `base` 的结果 |
| `composed(initial)` | `FilterComposed` | 以 and/or 逻辑组合多个过滤器 |
| `compile(code)` | `EventFilters$Compiled` | 把 Java 代码编译成过滤器（事件按 `BaseEvent` 处理） |
| `compile(event, code)` | `EventFilters$Compiled` | 针对具体事件类型编译，代码里能用该事件的字段 |
| `compile(event, ...conditions)` | `EventFilters$Compiled` | 多个条件自动以 `&&` 连接 |
| `getGlobalsForCompiled()` | `JavaMap` | 编译过滤器共享的全局 Map（d.ts 无进一步说明，进阶用途） |

此外类上有一个静态常量 `EventFilters.CONSTANT_TRUE`（恒真过滤器），需要通过完整类名访问：`Packages.xyz.wagyourtail.jsmacros.core.event.EventFilters.CONSTANT_TRUE`。

所有过滤器都实现 `EventFilter` 接口：`test(event)` 返回是否放行，`canFilter(event)` 返回是否适用于某事件名。

### modulus：降频

最常用的一个。`Tick` 每秒触发 20 次，套上 `modulus(20)` 就变成约每秒一次：

```javascript
const every20 = JsMacros.eventFilters().modulus(20)

const listener = JsMacros.on("Tick", every20, JavaWrapper.methodToJava(() => {
    Chat.actionbar(`时间: ${World.getTimeOfDay()}`)
}))
```

比在回调里自己维护计数器干净得多，而且被挡掉的 19 次连 JS 都不进。

`FilterModulus` 对象成员：`quotient`（字段，除数）、`count`（字段，内部计数）、`setQuotient(quotient)`（改除数，返回自身）。

### limited：只要前 N 次

```javascript
const firstThree = JsMacros.eventFilters().limited(3)

JsMacros.on("RecvMessage", firstThree, JavaWrapper.methodToJava((e) => {
    if (e.text) Chat.log(`前三条消息之一: ${e.text.getString()}`)
}))
```

`FilterLimited` 对象成员：`limit` / `count`（字段）、`setLimit(limit)`、`reset()`（计数清零，重新放行，返回自身）。

!!! note "次数用完监听器还在"
    `limited` 耗尽后监听器并不会注销，只是永远不再触发。不再需要就 `JsMacros.off(listener)` 摘掉；想"再来 N 次"则调用 `filter.reset()`。只触发一次的需求直接用 `JsMacros.once(...)` 更简单。

### invert：取反

反转任意过滤器的结果。d.ts 特别说明它会识别双重取反：`invert(invert(filter))` 就是原来的 `filter`。

```javascript
const f = JsMacros.eventFilters()

// limited(100) 放行前 100 次 → 取反后变成"跳过前 100 个 Tick(5 秒), 之后一直放行"
const afterWarmup = f.invert(f.limited(100))

JsMacros.on("Tick", afterWarmup, JavaWrapper.methodToJava(() => {
    // 进服 5 秒后才开始工作, 避开加载期的抖动
}))
```

### composed：组合

`composed(initial)` 以一个过滤器打底，然后链式 `and(filter)` / `or(filter)`（都返回自身，可继续链）：

```javascript
const f = JsMacros.eventFilters()

// 每 2 tick 一次, 且总共只要前 100 次
const combo = f.composed(f.modulus(2)).and(f.limited(100))

JsMacros.on("Tick", combo, JavaWrapper.methodToJava(() => {
    // 以 10 次/秒 的频率执行, 10 秒后彻底安静
}))
```

`FilterComposed` 其余成员：`test(event)`、`canFilter(event)`、`getChildren()`（子过滤器流）。

!!! tip "混用 and/or 时保持结构清晰"
    d.ts 没有说明链式混用 `and`/`or` 时的结合顺序。需要复杂逻辑时，用嵌套的 `composed(...)` 把括号结构显式写出来，别赌优先级。

### compile：Java 代码过滤器

`compile` 把一段 **Java 方法体**编译成过滤器，是高频事件过滤的终极手段——条件判断完全在 Java 侧完成。d.ts JSDoc 要点（翻译）：

- 代码里可用变量是 `event`（对应事件类型的对象）以及 `CompiledCommons` 的成员（d.ts 未列出完整清单；官方示例中出现了 `eq(a, b)`，用于相等比较）；
- 如果代码只有一行且末尾没有 `;`，会自动帮你补上 `return` 和 `;`；
- 多行代码要自己写 `return` 和分号。

官方示例（翻译自 JSDoc）：

```javascript
const f = JsMacros.eventFilters()

// 只放行方块更新包
const packetFilter = f.compile("RecvPacket", 'eq(event.type, "BlockUpdateS2CPacket")')

// 只放行 W 键按下
const keyFilter = f.compile("Key", 'event.action == 1 && eq(event.key, "key.keyboard.w")')

// 多行写法: 自己写 return 和分号
const soundFilter = f.compile("Sound", `
    float pitch = event.pitch;
    // 这里可以写任意多行 Java
    return pitch == 0.625f;
`)
```

多条件重载 `compile(event, ...conditions)` 会把条件用 `(条件1)&&(条件2)` 的方式连接：

```javascript
// 等价于 compile("Key", '(event.action == 1)&&(eq(event.key, "key.keyboard.g"))')
const gDown = f.compile("Key", "event.action == 1", 'eq(event.key, "key.keyboard.g")')

JsMacros.on("Key", gDown, JavaWrapper.methodToJava(() => Chat.log("按下了 G")))
```

不带事件名的 `compile(code)` 把事件当 `BaseEvent` 处理，拿不到具体字段，适合只依赖通用信息的过滤。

!!! warning "这是 Java，不是 JS"
    - 字符串比较不能用 `==`，用示例里的 `eq(...)`；
    - 数字有类型之分，比较浮点字段注意 `f` 后缀（如 `0.625f`）；
    - 写错了在创建过滤器时就会抛异常，调试成本比 JS 高。

!!! tip "先用简单的"
    大多数场景 `modulus` / `limited` / 普通回调判断就够了。只有确认性能真的成问题（通常是 `RecvPacket`/`SendPacket`），再上 `compile`。

## on 的参数顺序

`JsMacros.on` 带过滤器的两个重载：

```javascript
JsMacros.on("Tick", filter, callback)
JsMacros.on("SendMessage", filter, true, callback) // joined = true
```

!!! warning "filter 在 joined 前面"
    有过滤器时参数顺序是 `on(event, filter, joined, callback)`，不要写成 `on(event, joined, filter, callback)`。

## 实战：过滤高频数据包

`RecvPacket` 是最需要过滤器的事件——每个数据包都会触发。把类型判断放到 Java 侧，脚本只在关心的包到来时才被唤醒：

```javascript
const f = JsMacros.eventFilters()
const blockUpdate = f.compile("RecvPacket", 'eq(event.type, "BlockUpdateS2CPacket")')

const listener = JsMacros.on("RecvPacket", blockUpdate, JavaWrapper.methodToJava((e) => {
    Chat.log("收到一个方块更新包")
}))
```

对比一下不用过滤器的写法：`JsMacros.on("RecvPacket", callback)` 里再 `if (e.type === ...)`——每个数据包都要完整走一遍"进 JS 上下文 → 判断 → 返回"，繁忙服务器上这就是每秒几百次无谓的 JS 调用。

## 高频事件性能建议

1. **`RecvPacket` / `SendPacket`**：一定带过滤器，优先 `compile` 按 `event.type` 筛类型；
2. **`Tick` 轮询**：`modulus(n)` 降频，别让 20 次/秒的回调做只需 1 次/秒的事；
3. **不要为了过滤而 joined**：过滤本身不需要 `joined = true`，joined 只在要取消/修改事件时用（见[事件系统](events.md)）；
4. **有状态的过滤器不要共享实例**：`modulus` / `limited` 内部有 `count` 计数，一个实例挂到两个监听器上会共用计数，各建各的；
5. **`waitForEvent` 只有 JS 谓词可用**：它没有 `EventFilter` 重载，等待型逻辑本身低频，JS 谓词足够。

## 下一步

- [事件系统](events.md)——`on` / `once` / `waitForEvent` 全部重载与 joined 机制
- [服务脚本](services.md)——常驻监听 + 过滤器是服务的标准搭配
- [全部事件参考](events_reference.md)——各事件的字段，写 `compile` 条件前先查这里
