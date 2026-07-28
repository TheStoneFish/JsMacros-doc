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

## 全局类型别名机制

`JsMacros-2.1.0.d.ts` 的末尾（48980 行起）为几乎所有 Helper 声明了全局别名：

```typescript
type ItemStackHelper = Packages.xyz.wagyourtail.jsmacros.client.api.helper.inventory.ItemStackHelper;
type BlockPosHelper = Packages.xyz.wagyourtail.jsmacros.client.api.helper.world.BlockPosHelper;
type EntityHelper<T = any> = Packages.xyz.wagyourtail.jsmacros.client.api.helper.world.entity.EntityHelper<T>;
```

意思是：写类型标注时**不用写完整包名**，直接用 `ItemStackHelper` 这种短名字即可。这纯粹是类型层的便利，运行时不存在这些名字（运行时真实存在的是 `Packages.xyz...` 这棵包树）。

### JSDoc / TS 注解写法

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

/** @type {JavaList<EntityHelper>} */
const entities = World.getEntities(8)
```

ts 文件里直接把这些名字当类型用。另外，事件脚本里的全局 `event` 类型是宽泛的 `Events.BaseEvent`，用 `JsMacros.assertEvent` 可以同时完成"运行时断言 + TS 类型收窄"：

```javascript
JsMacros.assertEvent(event, 'RecvMessage')
// 从这行往下，编辑器就知道 event.text 存在了
```

## 常用类型速查表

| 类型 | 一句话 | 详见 |
| --- | --- | --- |
| `TextHelper` | Minecraft 文本组件（聊天消息、物品名），不是字符串 | [聊天与文本](chat.md) |
| `TextBuilder` | 构造带颜色/点击/悬停的文本 | [聊天与文本](chat.md) |
| `ItemStackHelper` | 一组物品（种类+数量+NBT） | [背包](inventory.md) |
| `Inventory` | 当前打开的背包/容器界面，负责点击槽位 | [背包](inventory.md) |
| `NBTElementHelper` | NBT 数据的包装 | [背包](inventory.md) |
| `BlockPosHelper` | 方块整数坐标，可 `up()`、`north()`、`distanceTo()` | [位置与向量](position.md) |
| `Pos2D` / `Pos3D` | 小数坐标，可 `add()`、`multiply()` | [位置与向量](position.md) |
| `Vec2D` / `Vec3D` | 两点构成的向量 | [位置与向量](position.md) |
| `BlockDataHelper` | 某个位置的完整方块信息（方块+状态+NBT） | [方块](blocks.md) |
| `BlockHelper` / `BlockStateHelper` | 方块类型 / 方块状态 | [方块](blocks.md) |
| `EntityHelper` | 任意实体的基类 | [实体](entities.md) |
| `LivingEntityHelper` | 有血量的实体（`getHealth()`） | [实体](entities.md) |
| `PlayerEntityHelper` | 玩家实体 | [实体](entities.md) |
| `ClientPlayerEntityHelper` | 你自己（`Player.getPlayer()` 的返回值） | [玩家](player.md) |
| `ChunkHelper` | 区块 | [世界与方块](world.md) |
| `Draw2D` / `Draw3D` | HUD 的 2D / 3D 画布 | [HUD 渲染](hud.md) |
| `IScreen` / `ScriptScreen` | 自定义 GUI 屏幕 | [脚本屏幕](screen.md) |
| `OptionsHelper` | 游戏设置读写 | [游戏设置](options.md) |
| `MethodWrapper` | `JavaWrapper.methodToJava()` 包装后的回调 | [脚本上下文](script_context.md) |
| `EventContainer` | 脚本上下文句柄（全局 `context` 的类型） | [脚本上下文](script_context.md) |
| `IEventListener` | `JsMacros.on()` 的返回值，用于 `off()` | [事件系统](events.md) |
| `FileHandler` | `FS.open()` 返回的文件句柄 | [文件系统](fs.md) |
| `HTTPRequest$Response` / `Websocket` | HTTP 响应 / WebSocket 连接 | [网络请求](network.md) |
| `ServerInfoHelper` | 服务器条目信息（ping 结果等） | [客户端](client.md) |

## JS 与 Java 类型互转

### Java 集合：JavaList / JavaMap / JavaSet / JavaArray

| 类型 | 类似 JS | 常用方法 |
| --- | --- | --- |
| `JavaList<T>` | `Array<T>` | `size()`、`get(i)`、`add(v)`、`contains(v)` |
| `JavaMap<K, V>` | `Map<K, V>` | `get(k)`、`put(k, v)`、`keySet()`、`containsKey(k)` |
| `JavaSet<T>` | `Set<T>` | `contains(v)`、`size()` |
| `JavaArray<T>` | 数组 | `length` 属性、下标访问 |

它们**不是** JS 数组：没有 `.map()`、`.filter()`、`.forEach()`。遍历方式：

```javascript
const entities = World.getEntities(8)
if (entities) {
    // 写法一：经典下标
    for (let i = 0; i < entities.size(); i++) {
        Chat.log(entities.get(i).getName().getString())
    }

    // 写法二：GraalJS 下 Java 集合一般可以直接 for...of
    for (const e of entities) {
        Chat.log(e.getName().getString())
    }
}
```

遍历 `JavaMap` 走 `keySet()`：

```javascript
const map = Player.openInventory().getMap()
for (const section of map.keySet()) {
    Chat.log(`${section}: ${map.get(section).length} 个槽位`)
}
```

想把 Java 集合转成真正的 JS 数组再用数组方法，手动倒一遍即可：

```javascript
const arr = []
for (const e of entities) arr.push(e)
const names = arr.map(e => e.getName().getString())
```

### JS 数组转 Java 集合

需要传"真正的 Java 集合"给 API 时，用 `JavaUtils`：

```javascript
const list = JavaUtils.createArrayList(["a", "b", "c"]) // java.util.ArrayList
const map = JavaUtils.createHashMap()                    // java.util.HashMap
map.put("Content-Type", "application/json")
```

### 数字：number 与 int / long / double

d.ts 签名里的 `int`、`long`、`float`、`double` 都是 Java 基本类型，JS 这边统一用 `number`，传参时自动转换。注意两点：

- 参数要求 `int` 时传整数，计算结果可能带小数的先 `Math.floor()`；
- `long` 的取值范围超过 JS `number` 的安全整数（2^53），毫秒时间戳没问题，但极大的 long 值可能丢精度。

### null 与 undefined

Java 世界只有 `null`，没有 `undefined`。d.ts 里标了 `| null` 的返回值（如 `Player.getPlayer()`、`GlobalVars.getBoolean()`）都要判空。判断时推荐宽松写法，一次覆盖两者：

```javascript
const p = Player.getPlayer()
if (p == null) { /* 两个等号：null 和 undefined 都拦住 */ }
if (!p) { /* 更简短，注意 0/""/false 也会命中 */ }
```

### 字符串字面量类型

d.ts 还定义了大量"合法取值就这几个"的字符串/数字类型，编辑器会直接提示可选值：

```typescript
type Direction = 'up' | 'down' | 'north' | 'south' | 'east' | 'west'
type HotbarSlot = 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 // d.ts 里以 Octit | 8 的形式定义，等价展开
type OffhandSlot = 40
```

常见的几组：

- **背包分区名**（`Inventory.getMap()` 的键，来自 `InvMapType` 命名空间）：`hotbar`、`main`、`offhand`、`container`、`input`、`output`、`fuel`、`helmet` / `chestplate` / `leggings` / `boots` 等，详见[背包](inventory.md)；
- **屏幕名**（`Inventory.getType()` 的返回值 `HandledScreenName`）：`Survival Inventory`、`1 Row Chest` ~ `9 Row Chest`、`Shulker Box`、`Furnace`、`Crafting Table`、`Anvil`、`Villager`、`Horse` 等；
- **方向**：`Direction`（六个面），用于 `interactBlock`、`BlockPosHelper.offset()` 等。

### 颜色

HUD 和发光颜色通常是整数 RGB / ARGB，直接用十六进制字面量最直观：

```javascript
const red = 0xFF0000
const green = 0x00FF00
const white = 0xFFFFFF
```

某些方法单独接收 `alpha` 参数，范围一般是 `0` 到 `255`；带透明度的 ARGB 写法形如 `0x80FF0000`（半透明红）。

## getRaw()：原始对象的诱惑与风险

每个 Helper 都能 `getRaw()` 拿到被包装的原始 Minecraft 对象：

```javascript
const rawPlayer = Player.getPlayer().getRaw()
```

风险要想清楚再用：

- **名字对不上**：d.ts 注释里的 `net.minecraft.*` 类名和方法名是开发映射下的名字，正式游戏环境里这些类是混淆过的（中间名），直接按源码里的方法名调用很可能找不到方法；
- **跨版本碎一地**：原始对象的结构随 Minecraft 版本变化，Helper 层会帮你兜住，raw 层不会；
- **绕过安全包装**：Helper 有判空和线程方面的处理，raw 对象没有。

结论：`getRaw()` 是逃生舱，不是日常通道。确实需要触碰原始对象（配合反射、调其他 Mod 的 API）时，先读 [Java 与反射](java_api.md)。

## 下一步

- 想看每个全局库的方法签名 → [全局库参考](global_api_reference.md)
- 想理解 `MethodWrapper`、线程和 joined → [脚本上下文](script_context.md)
- 回到地图 → [API 总览](api_overview.md)
