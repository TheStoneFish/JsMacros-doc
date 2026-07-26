---
icon: lucide/scan-search
---

# 世界扫描器（WorldScanner）

`WorldScanner` 是 JsMacros 提供的高性能方块搜索器。相比 [`World.findBlocksMatching`](world.md#findblocksmatching)，它有三大优势：

1. **可复用**：过滤条件只定义一次，之后可以反复扫描不同区域（找矿 ESP 每秒刷新时差距巨大）。
2. **更快**：用 Builder 创建的扫描器，过滤逻辑完全运行在 Java 侧，可以用并行流多线程扫描——JavaScript 本身是单线程的，`MethodWrapper` 回调过滤只能逐个方块顺序执行，Builder 正是为了解决这个问题而设计的。
3. **更灵活**：任意组合方块/状态条件（硬度、光照、朝向、名字包含……），支持按区块范围、立方体、球体、玩家触及范围等多种区域扫描，还能统计区块内方块数量。

## 两种创建方式

### 方式一：Builder 链式（推荐）

`World.getWorldScanner()` 不带参数时返回一个 `WorldScannerBuilder`，链式声明条件后 `build()`：

```javascript
const scanner = World.getWorldScanner()
    .withStringBlockFilter().contains("diamond_ore")   // 方块 ID 里含 diamond_ore
    .build()

const found = scanner.scanAroundPlayer(2)
Chat.log(`附近钻石矿: ${found.size()} 个`)
```

d.ts 里的官方例子——"朝南、不需要工具、硬度不超过 10、名字含 chest 或 barrel 的方块"：

```javascript
const scanner = World.getWorldScanner()
    .withBlockFilter("getHardness").is("<=", 10)
    .andStringBlockFilter().contains("chest", "barrel")
    .withStringStateFilter().contains("facing=south")
    .andStateFilter("isToolRequired").is(false)
    .build()
```

### 方式二：MethodWrapper 回调

`World.getWorldScanner(blockFilter, stateFilter)` 直接返回 `WorldScanner`（世界未加载时为 `null`）。两个过滤器分别收到 [`BlockHelper`](blocks.md)（方块类型，与位置无关）和 [`BlockStateHelper`](blocks.md)（方块状态），返回 `true` 表示匹配；不需要的一侧传 `null`，两侧都给时须**同时**通过：

```javascript
const scanner = World.getWorldScanner(
    JavaWrapper.methodToJava((block) => block.getId() === "minecraft:spawner"),
    null   // 不过滤方块状态
)
if (scanner) {
    Chat.log(`附近刷怪笼: ${scanner.scanAroundPlayer(4).size()} 个`)
}
```

!!! warning "回调版更慢"
    每个候选方块都要从 Java 跨进 JavaScript 调一次回调，而且强制单线程顺序扫描。条件能用 Builder 表达就用 Builder，回调版留给真正复杂的逻辑。

## WorldScannerBuilder 方法全表

Builder 有**方块过滤器**（block filter）和**状态过滤器**（state filter）两条独立的过滤链，各自单独配置，没配置的那条直接忽略。

### 开启与接续过滤器

| 方法 | 作用 |
| --- | --- |
| `withBlockFilter(method)` | 开启方块过滤链，检查 `BlockHelper` 上名为 `method` 的方法的返回值（如 `"getHardness"`） |
| `withStateFilter(method)` | 开启状态过滤链，检查 `BlockStateHelper` 上的方法（如 `"isToolRequired"`） |
| `withStringBlockFilter()` | 开启方块过滤链，按方块的字符串形式（`toString()`）比较 |
| `withStringStateFilter()` | 开启状态过滤链，按状态的字符串形式比较 |
| `andBlockFilter(method)` / `andStateFilter(method)` | 以**与**逻辑追加一个方法条件 |
| `orBlockFilter(method)` / `orStateFilter(method)` | 以**或**逻辑追加一个方法条件 |
| `andStringBlockFilter()` / `orStringBlockFilter()` | 以与 / 或逻辑追加字符串条件（方块链） |
| `andStringStateFilter()` / `orStringStateFilter()` | 以与 / 或逻辑追加字符串条件（状态链） |
| `notBlockFilter()` / `notStateFilter()` | 把**整条**方块 / 状态过滤链取反，不需要参数 |

### 谓词（紧跟在过滤器声明之后）

| 方法 | 作用 |
| --- | --- |
| `is(...args)` | 比较上一步方法的返回值，参数形式取决于返回值类型（见下） |
| `is(methodArgs, filterArgs)` | 双数组形式：`methodArgs` 传给被调用的方法，`filterArgs` 是比较条件 |
| `test(...args)` / `test(methodArgs, filterArgs)` | 与 `is` 完全等价（`is` 在某些语言里是关键字） |
| `equals(...strings)` | 字符串过滤器：等于任一参数 |
| `contains(...strings)` | 字符串过滤器：包含任一参数 |
| `startsWith(...strings)` | 字符串过滤器：以任一参数开头 |
| `endsWith(...strings)` | 字符串过滤器：以任一参数结尾 |
| `matches(...strings)` | 字符串过滤器：匹配任一正则表达式 |
| `build()` | 生成 `WorldScanner` |

### 链式组合逻辑

- 一条过滤链必须以 `with...` 开头；再写一次 `with...` 会**覆盖**同类型的整条旧链。
- 追加条件用 `and.../or...` 前缀；`not...` 把整条链取反。
- 每个 `with/and/or` 后面**必须紧跟一个谓词**（`is`/`test` 或字符串谓词）才算一个完整条件。
- 字符串谓词接受多个参数，参数之间是**或**的关系：`contains("chest", "barrel")` 表示含 chest 或含 barrel。

`is`/`test` 的参数规则，取决于所查方法的返回类型：

| 返回类型 | 写法 | 例子 |
| --- | --- | --- |
| 数字 | `is(op, 数字)`，`op` 可为 `">"`、`">="`、`"<"`、`"<="`、`"=="`、`"!="` | `withBlockFilter("getHardness").is(">=", 8)` |
| 字符串 | `is(模式, 字符串)`，模式可为 `"EQUALS"`、`"CONTAINS"`、`"STARTS_WITH"`、`"ENDS_WITH"`、`"MATCHES"` | `withBlockFilter("getId").is("ENDS_WITH", "ore")` |
| 布尔 | `is(布尔值)` | `withStateFilter("isToolRequired").is(false)` |

!!! warning "字符串过滤器的 toString 陷阱"
    `withStringBlockFilter()` 比较的是方块的 `toString()` 结果，形如 `{minecraft:stone}`（带花括号），所以 `equals("minecraft:stone")` **永远是 false**。要么用 `contains("minecraft:stone")`，要么改用精确匹配：`withBlockFilter("getId").is("EQUALS", "minecraft:stone")`。

## WorldScanner 方法全表

所有扫描方法都返回 `JavaList<Pos3D>`（匹配方块的坐标列表），坐标取整数用 `pos.x` / `pos.y` / `pos.z` 字段。

### 按区块扫描

| 方法 | 说明 |
| --- | --- |
| `scanAroundPlayer(chunkRange)` | 以玩家所在区块为中心扫描。`0` 只扫当前区块，`1` 扫 3×3 区块，边长为 `2*range+1` |
| `scanChunkRange(centerX, centerZ, chunkrange)` | 以指定**区块坐标**为中心扫描 |
| `getChunkRange(centerX, centerZ, chunkrange)` | 只返回范围内的区块坐标列表（原始 `ChunkPos`），不扫方块 |

### 按方块区域扫描（1.9.1+）

| 方法 | 说明 |
| --- | --- |
| `scanCubeArea(pos, range)` | 以 `BlockPosHelper` 为中心、`range` 为半径的立方体 |
| `scanCubeArea(x, y, z, range)` | 同上，坐标形式 |
| `scanCubeArea(pos1, pos2)` | 两点围成的长方体，`pos1` 含、`pos2` **不含** |
| `scanCubeArea(x1, y1, z1, x2, y2, z2)` | 同上，坐标形式（第二组坐标不含） |
| `scanCubeAreaInclusive(pos1, pos2)` | 两点围成的长方体，两端**都含** |
| `scanCubeAreaInclusive(x1, y1, z1, x2, y2, z2)` | 同上，坐标形式 |
| `scanSphereArea(pos, radius)` | 以 `Pos3D` 为球心的球体 |
| `scanSphereArea(x, y, z, radius)` | 同上，坐标形式 |

### 玩家触及范围扫描（1.9.1+）

找"够得着"的目标方块（自动挖矿/自动放置类脚本常用）。注意这组方法**不检查视线遮挡**，只按距离算：

| 方法 | 说明 |
| --- | --- |
| `scanReachable()` | 以玩家眼睛位置和当前触及距离扫描 |
| `scanReachable(strict)` | `strict = true`（默认）按方块轮廓判定，`false` 按完整立方体 |
| `scanReachable(pos)` | 指定中心点，触及距离取玩家的 |
| `scanReachable(pos, reach)` | 指定中心点和触及距离 |
| `scanReachable(pos, reach, strict)` | 全参数版；`pos` 一般传 `Player.getPlayer().getEyePos()`，`reach` 传 `Player.getInteractionManager().getReach()` |
| `scanClosestReachable()` | 只返回最近的一个（`Pos3D` 或 `null`） |
| `scanClosestReachable(strict)` | 同上，指定判定方式 |
| `scanClosestReachable(pos, reach, strict)` | 同上，全参数版 |

### 统计与缓存

| 方法 | 说明 |
| --- | --- |
| `getBlocksInChunk(chunkX, chunkZ, ignoreState)` | 统计单个区块内匹配方块的数量，返回 `JavaMap<string, number>`；`ignoreState = true` 时同方块的不同状态合并计数 |
| `getBlocksInChunks(centerX, centerZ, chunkRange, ignoreState)` | 统计一片区块范围内的数量 |
| `getCachedAmount()` | 已缓存的方块状态判定数（正常在 200~400 左右） |

```javascript
// 数一数当前区块里各种矿有多少
const oreScanner = World.getWorldScanner()
    .withBlockFilter("getId").is("ENDS_WITH", "_ore")
    .build()
const p = Player.getPlayer().getBlockPos()
const counts = oreScanner.getBlocksInChunk(p.getX() >> 4, p.getZ() >> 4, true)
for (const id of counts.keySet()) {
    Chat.log(`${id}: ${counts.get(id)}`)
}
```

## 完整示例：钻石矿 ESP

扫描玩家周围 5 区块内的钻石矿（普通 + 深板岩），用 [`Draw3D`](hud.md) 画出穿墙可见的高亮框，显示 30 秒后自动清除。绑到按键直接可用：

```javascript
// 1. 建扫描器：ID 含 diamond_ore 即可同时命中两种钻石矿
const scanner = World.getWorldScanner()
    .withStringBlockFilter().contains("diamond_ore")
    .build()

// 2. 扫描周围 5 区块（11x11 区块，含全部高度）
const positions = scanner.scanAroundPlayer(5)
Chat.log(`找到 ${positions.size()} 个钻石矿`)

// 3. 画 ESP 框
const d3d = Hud.createDraw3D()
for (const pos of positions) {
    d3d.addBox(
        pos.x, pos.y, pos.z,
        pos.x + 1, pos.y + 1, pos.z + 1,
        0x55FFFF, 255,   // 边框颜色、不透明度
        0x55FFFF, 64,    // 填充颜色、不透明度
        true,            // 填充
        false            // false = 不做深度剔除，隔墙也能看见
    )
}
d3d.register()

// 4. 30 秒后清除
Client.waitTick(20 * 30)
d3d.unregister()
```

!!! tip "想要持续刷新?"
    把扫描 + 重画包进循环或 `Tick` 事件即可，但**扫描器只建一次**（`build()` 放循环外），每轮先 `d3d.clear()` 再重新 `addBox`。长期运行的 ESP 建议做成[服务](services.md)，用 `event.unregisterOnStop(true, d3d)` 保证停止时清屏。

## 性能与线程注意

!!! warning "扫描是阻塞调用"
    `scanAroundPlayer` 等方法在**当前脚本线程**上同步执行，范围越大越久（区块数按 `(2n+1)²` 增长）。键绑定、聊天命令触发的脚本跑在独立线程，扫几秒只是脚本变慢；但**不要**在会阻塞主线程的场合（如需要 `joined` 的事件回调）里做大范围扫描。

- **Builder 优先**：Builder 生成的过滤器是纯 Java 函数，扫描时走并行流；`MethodWrapper` 回调版强制顺序执行，还有每方块一次的跨语言开销。
- **复用扫描器**：扫描器会缓存方块状态的判定结果（`getCachedAmount()` 可查，通常 200~400 条），同一个扫描器反复用比每次重建快得多。
- **从小范围开始**：`chunkRange` 先试 `1`~`2`；找矿类需求一般 `5` 以内足够，`10` 就是 441 个区块了。
- **结果只是坐标**：返回的 `Pos3D` 不含方块信息，需要时再 `World.getBlock(pos.x, pos.y, pos.z)` 查详情（注意扫描结果可能已过时，方块可能被挖掉）。
