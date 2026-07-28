---
icon: lucide/braces
---

# 常用类型

JsMacros 脚本运行在 JVM 上，代码里会混用三种东西：普通 JS 值、Java 对象、以及 JsMacros 自己的 Helper 包装。理解这套类型系统，比死背方法更重要——它能解释"为什么这个返回值不能当字符串用"、"为什么这个列表没有 forEach"。

## Helper 包装哲学

Minecraft 内部的对象（玩家、物品、方块）是原始 Java 对象，方法名混乱、跨版本变动大。JsMacros 的做法是给它们套一层 **Helper**：

```
原始 Minecraft 对象          JsMacros 包装
net.minecraft...ItemStack  →  ItemStackHelper
net.minecraft...Entity     →  EntityHelper
net.minecraft...Component  →  TextHelper
BlockPos                   →  BlockPosHelper
```

所有 Helper 继承自 `BaseHelper<T>`，它只有一个通用方法：`getRaw()` 返回被包装的原始对象（风险见本页末尾）。你平时从 API 拿到的几乎都是 Helper——方法名稳定、带文档、跨版本兼容，**能用 Helper 就别碰原始对象**。

反方向也有工具：`JavaUtils.getHelperFromRaw(raw)` 尝试把一个原始对象包装回正确的 Helper。
### TS 注解写法

配合[快速开始](quick_start.md)里的开发环境，js 文件里用 JSDoc 就能拿到精确补全：

```javascript
/** @type {ItemStackHelper} */
const held = Player.openInventory().getHeld()

/**
 * @param {EntityHelper} entity
 * @param {number} maxDist
 * @returns {boolean}
 */
function isNear(entity, maxDist) {
    const p = Player.getPlayer()
    return p != null && entity.distanceTo(p) <= maxDist
}

```
ts 文件里直接把这些名字当类型用。另外，事件脚本里的全局 `event` 类型是宽泛的 `Events.BaseEvent`，用 `JsMacros.assertEvent` 可以同时完成"运行时断言 + TS 类型收窄"：

```javascript
JsMacros.assertEvent(event, 'RecvMessage')
// 从这行往下，编辑器就知道 event.text 存在了, 发现完全可以补全成功
Chat.log(event.text)
```
## Java类型与Js类型

!!! note "运行时的类型并不总是「纯 Java」或「纯 JS」"
    JsMacros（GraalJS 等）会对 Java 对象做**互操作包装**。同一个对象上，往往既能调 Java 方法，也能在一定程度上当 JS 对象用，

### JS 数组转 Java 集合

需要传"真正的 Java 集合"给 API 时，用 `JavaUtils`：

```javascript
// java.util.ArrayList
const list = JavaUtils.createArrayList(["a", "b", "c"])
// java.util.HashMap
const map = JavaUtils.createHashMap()                    
map.put("Content-Type", "application/json")
```

### number 与 int / long / double

`int`、`long`、`float`、`double` 都是 Java 基本类型，JS 这边统一用 `number`，传参时自动转换。注意两点：

- 参数要求 `int` 时传整数，计算结果可能带小数的先 `Math.floor()`；
- `long` 的取值范围超过 JS `number` 的安全整数（2^53），毫秒时间戳没问题，但极大的 long 值可能丢精度。

### null

Java 世界只有 `null`，没有 `undefined`, 有 `null` 的返回值（如 `Player.getPlayer()`、`GlobalVars.getBoolean()`）都要判空。判断时推荐宽松写法，一次覆盖两者：

```javascript
const p = Player.getPlayer()
if (p == null) { /* 两个等号：null 和 undefined 都拦住 */ }
if (!p) { /* 更简短，注意 0/""/false 也会命中 */ }
```

## getRaw()原始对象的风险

每个 Helper 都能 `getRaw()` 拿到被包装的原始 Minecraft 对象：

```javascript
const rawPlayer = Player.getPlayer().getRaw()
```

风险要想清楚再用：

- **名字对不上**：d.ts 注释里的 `net.minecraft.*` 类名和方法名是开发映射下的名字，正式游戏环境里这些类是混淆过的（中间名），直接按源码里的方法名调用很可能找不到方法；
- **跨版本碎一地**：原始对象的结构随 Minecraft 版本变化，Helper 层会帮你兜住，raw 层不会；
- **绕过安全包装**：Helper 有判空和线程方面的处理，raw 对象没有。

结论：`getRaw()` 是逃生舱，不是日常通道。确实需要触碰原始对象（配合反射、调其他 Mod 的 API）时，先读 [Java 与反射](java_api.md)。
