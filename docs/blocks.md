---
icon: lucide/box
---

# 方块 Helper

在 JsMacros 里操作方块，几乎都绕不开这一组 Helper 类：`BlockDataHelper`、`BlockHelper`、`BlockStateHelper`、`BlockPosHelper`，以及它们的"周边"——`UniversalBlockStateHelper`、`FluidStateHelper`、`DirectionHelper`、`ChunkHelper` 和 `HitResultHelper`。本页把它们逐个讲清楚：是什么、从哪来、有哪些方法、怎么用。

## 三层概念：先分清这几个类型

| 类型 | 表示什么 | 一句话记忆 |
| --- | --- | --- |
| `BlockDataHelper` | **某个坐标处**的方块实例 | 位置 + 状态 + NBT 的打包 |
| `BlockHelper` | 方块**类型**定义 | "石头"这个种类本身，不绑定坐标 |
| `BlockStateHelper` | 方块**状态** | 朝向、是否点亮、含水等属性组合 |
| `BlockPosHelper` | 整数方块坐标 | 就是一个 (x, y, z) |

三者的关系：

```text
BlockDataHelper（世界中 (x, y, z) 处的那个方块）
 ├── getBlockPos()          → BlockPosHelper（它在哪）
 ├── getBlockStateHelper()  → BlockStateHelper（它现在什么状态）
 │        └── getBlock()    → BlockHelper（它是什么方块）
 └── getNBT()               → 方块实体 NBT（箱子内容、告示牌文字等，可能为 null）
```

### 这些对象从哪来？

| 来源 | 得到的类型 | 说明 |
| --- | --- | --- |
| `World.getBlock(x, y, z)` / `World.getBlock(pos)` | `BlockDataHelper \| null` | 最常用入口，区块未加载时返回 `null` |
| `World.iterateSphere(...)` / `World.iterateBox(...)` | 回调收到 `BlockDataHelper` | 遍历一片区域 |
| `World.findBlocksMatching(...)` | `JavaList<Pos3D>` | 只返回坐标列表，需再用 `World.getBlock(pos)` 取方块 |
| `Player.rayTraceBlock(distance, fluid)` | `BlockDataHelper \| null` | 准星射线命中的方块 |
| `Player.detailedRayTraceBlock(distance, fluid)` | `HitResultHelper$Block` | 带命中面信息的射线结果 |
| 事件字段，如 `AttackBlock`、`InteractBlock`、`BlockUpdate` 的 `event.block` | `BlockDataHelper` | 事件触发时直接拿到 |
| `Registries.getBlock(id)` | `BlockHelper` | 按 ID 查方块类型，不需要世界中真的存在 |
| `World.getChunk(x, z)` | `ChunkHelper \| null` | 整个区块 |

!!! tip "别混 ID 和显示名"
    `getId()` 返回的才是脚本判断用的 `minecraft:stone`；`getName()` 返回 `TextHelper`，是给人看的本地化文本（如"石头"），要显示时再 `.getString()`。

## BlockDataHelper：某坐标处的方块

```javascript
const block = World.getBlock(0, 64, 0)
if (block) {
    Chat.log(block.getId())                    // minecraft:stone
    Chat.log(block.getName().getString())      // 石头
    Chat.log(`${block.getX()}, ${block.getY()}, ${block.getZ()}`)
}
```

全部方法：

| 方法 | 返回类型 | 作用 |
| --- | --- | --- |
| `getX()` / `getY()` / `getZ()` | `number` | 方块坐标 |
| `getId()` | `string` | 方块 ID，如 `minecraft:chest` |
| `getName()` | `TextHelper` | 本地化显示名 |
| `getNBT()` | `NBTCompoundHelper \| null` | 方块实体 NBT，普通方块（石头、泥土）为 `null` |
| `getBlockStateHelper()` | `BlockStateHelper` | 方块状态 helper |
| `getBlock()` | `BlockHelper` | 方块类型 helper |
| `getBlockHelper()` | `BlockHelper` | 同上，**已废弃**，请改用 `getBlock()` |
| `getBlockState()` | `JavaMap<string, string>` | 状态的键值映射，如 `{facing: north, open: false}` |
| `getBlockPos()` | `BlockPosHelper` | 坐标 helper |
| `getRawBlock()` / `getRawBlockState()` / `getRawBlockEntity()` | Java 原始对象 | Minecraft 底层对象，高级用法才需要 |

!!! warning "记得判空"
    `World.getBlock(...)` 在区块未加载（离玩家太远）时返回 `null`，用前先判断。`getNBT()` 只有箱子、告示牌、熔炉这类**有方块实体**的方块才非空。

## BlockHelper：方块类型

`BlockHelper` 描述"这是什么方块"，与坐标无关。同一种方块在世界任何位置拿到的 `BlockHelper` 属性都一样。

| 方法 | 返回类型 | 作用 |
| --- | --- | --- |
| `getId()` | `string` | 方块 ID |
| `getName()` | `TextHelper` | 本地化显示名 |
| `getDefaultState()` | `BlockStateHelper` | 默认方块状态 |
| `getDefaultItemStack()` | `ItemStackHelper` | 对应的物品堆 |
| `getStates()` | `JavaList<BlockStateHelper>` | 该方块所有可能的状态 |
| `getTags()` | `JavaList<string>` | 方块标签 ID 列表，如 `minecraft:mineable/pickaxe` |
| `getHardness()` | `number` | 硬度（挖掘耗时基准），基岩为 `-1` |
| `getBlastResistance()` | `number` | 爆炸抗性 |
| `getSlipperiness()` | `number` | 滑度（冰 > 0.6） |
| `getJumpVelocityMultiplier()` | `number` | 跳跃速度倍率（蜂蜜块 < 1） |
| `getVelocityMultiplier()` | `number` | 移动速度倍率（灵魂沙 < 1） |
| `canMobSpawnInside()` | `boolean` | 生物能否在其内部生成 |
| `hasDynamicBounds()` | `boolean` | 是否有动态碰撞箱 |

```javascript
const helper = World.getBlock(0, 64, 0).getBlock()
Chat.log(`硬度=${helper.getHardness()}, 爆炸抗性=${helper.getBlastResistance()}`)

// 不需要世界中真的有这个方块，也能查类型信息
const obsidian = Registries.getBlock("minecraft:obsidian")
Chat.log(obsidian.getName().getString())  // 黑曜石
```

## BlockPosHelper：整数方块坐标

### 从哪来

```javascript
const pos1 = Player.getPlayer().getBlockPos()              // 玩家脚下坐标
const pos2 = World.getBlock(0, 64, 0).getBlockPos()        // 从方块拿
const pos3 = PositionCommon.createBlockPos(100, 64, -200)  // 手动创建
```

### 全部方法

**读取坐标**

| 方法 | 返回类型 | 作用 |
| --- | --- | --- |
| `getX()` / `getY()` / `getZ()` | `number` | 三个坐标分量 |

**偏移（都返回新的 `BlockPosHelper`，原对象不变）**

| 方法 | 作用 |
| --- | --- |
| `up()` / `up(distance)` | 向上 1 格 / n 格 |
| `down()` / `down(distance)` | 向下 1 格 / n 格 |
| `north()` / `north(distance)` | 向北（-Z）1 格 / n 格 |
| `south()` / `south(distance)` | 向南（+Z）1 格 / n 格 |
| `east()` / `east(distance)` | 向东（+X）1 格 / n 格 |
| `west()` / `west(distance)` | 向西（-X）1 格 / n 格 |
| `offset(direction)` | 按方向名偏移 1 格，方向为 `"down"` / `"up"` / `"north"` / `"south"` / `"west"` / `"east"` |
| `offset(direction, distance)` | 按方向名偏移 n 格 |
| `offset(x, y, z)` | 按三个分量偏移 |

**转换与距离**

| 方法 | 返回类型 | 作用 |
| --- | --- | --- |
| `toPos3D()` | `Pos3D` | 转成小数坐标向量，见[位置与向量](position.md) |
| `toNetherCoords()` | `BlockPosHelper` | 换算成对应的下界坐标（x、z 除以 8） |
| `toOverworldCoords()` | `BlockPosHelper` | 换算成对应的主世界坐标（x、z 乘以 8） |
| `distanceTo(entity)` | `number` | 到实体的距离 |
| `distanceTo(pos: BlockPosHelper)` | `number` | 到另一个方块坐标的距离 |
| `distanceTo(pos: Pos3D)` | `number` | 到小数坐标的距离 |
| `distanceTo(x, y, z)` | `number` | 到指定坐标的距离 |

### 示例：找脚下方块、按朝向偏移

```javascript
const player = Player.getPlayer()
const feet = player.getBlockPos()

// 脚下那格（站立面）
const ground = World.getBlock(feet.down())
if (ground) Chat.log(`你踩着: ${ground.getName().getString()}`)

// 面前 3 格的方块（按玩家水平朝向偏移）
const facing = player.getFacingDirection().getName()   // 如 "north"
const front = World.getBlock(feet.offset(facing, 3))
if (front) Chat.log(`面前 3 格: ${front.getName().getString()}`)
```

!!! note "偏移不改变原对象"
    `pos.up()` 返回一个**新**坐标，`pos` 本身不变，所以可以放心链式调用：`pos.up(2).north().east(5)`。

## BlockStateHelper：方块状态

方块状态 = 方块类型 + 一组属性值。比如同一扇门，有"开/关""上半/下半""朝向"等不同状态。

```javascript
const state = World.getBlock(0, 64, 0).getBlockStateHelper()
Chat.log(state.isAir())
Chat.log(state.isReplaceable())
Chat.log(state.toMap().toString())   // {snowy=false} 之类的属性表
```

全部方法：

**基本信息**

| 方法 | 返回类型 | 作用 |
| --- | --- | --- |
| `getBlock()` | `BlockHelper` | 该状态所属的方块类型 |
| `getId()` | `string` | 方块 ID |
| `getFluidState()` | `FluidStateHelper` | 该状态的流体信息（含水方块、岩浆等） |
| `getHardness()` | `number` | 当前状态的硬度 |
| `getLuminance()` | `number` | 发光等级 0~15（火把 14、萤石 15） |
| `getPistonBehaviour()` | `string` | 活塞行为：`NORMAL` / `DESTROY` / `BLOCK` / `IGNORE` / `PUSH_ONLY` |

**布尔判断**

| 方法 | 作用 |
| --- | --- |
| `isAir()` | 是否空气 |
| `isOpaque()` | 是否不透明 |
| `isSolid()` | 是否实体方块 |
| `isLiquid()` | 是否液体 |
| `isReplaceable()` | 是否可直接被替换（空气、草、水等，放方块时不用先敲掉） |
| `isBurnable()` | 是否可燃 |
| `isToolRequired()` | 挖掘掉落是否**必须**用对应工具（如铁矿必须镐） |
| `blocksMovement()` | 是否阻挡实体移动 |
| `emitsRedstonePower()` | 是否发出红石信号 |
| `exceedsCube()` | 形状是否超出一格立方体 |
| `hasBlockEntity()` | 是否有方块实体（箱子、告示牌等） |
| `hasRandomTicks()` | 是否接受随机刻（作物生长等） |
| `hasComparatorOutput()` | 是否有比较器输出 |
| `allowsSpawning(pos, entityId)` | 指定实体能否在此状态上生成 |
| `shouldSuffocate(pos)` | 实体在其中是否窒息 |

**进阶入口**

| 方法 | 返回类型 | 作用 |
| --- | --- | --- |
| `getUniversal()` | `UniversalBlockStateHelper` | 读取任意状态属性的万能入口，见下节 |
| `toMap()` | `JavaMap<string, string>` | 所有属性的键值映射（继承自 `StateHelper`） |
| `with(property, value)` | `StateHelper` | 生成把某属性改为指定值的新状态（继承自 `StateHelper`） |

## UniversalBlockStateHelper：读取任意状态属性

`BlockStateHelper` 只提供通用判断，而"这扇门开没开""这块小麦几级了"这类**具体属性**要靠 `UniversalBlockStateHelper`。它继承自 `BlockStateHelper`，把原版几乎所有方块状态属性都做成了 getter（一百多个方法），通过 `getUniversal()` 获得：

```javascript
const state = World.getBlock(x, y, z).getBlockStateHelper()
const uni = state.getUniversal()
```

代表性方法按功能分组：

| 分组 | 代表性方法 | 适用方块举例 |
| --- | --- | --- |
| 朝向 / 轴 | `getFacing()`、`getHorizontalFacing()`、`getHopperFacing()`（返回 `DirectionHelper`）；`getAxis()`、`getHorizontalAxis()`、`getOrientation()`、`getBlockFace()`；`isUp()` / `isDown()` / `isNorth()` / `isSouth()` / `isEast()` / `isWest()` | 熔炉、漏斗、原木、藤蔓 |
| 红石 / 机械 | `isPowered()`、`getPower()`、`isLit()`、`isLocked()`、`getDelay()`、`getComparatorMode()`、`isExtended()`、`isTriggered()`、`isConditional()`、`isEnabled()`、`isInverted()`、`getNote()`、`getInstrument()`、`get*WireConnection()` | 中继器、比较器、活塞、发射器、音符盒、红石线 |
| 开合 / 门窗 | `isOpen()`、`getDoorHinge()`、`isInWall()`、`getBlockHalf()`、`getDoubleBlockHalf()` | 门、活板门、栅栏门、桶 |
| 植物 / 生长 | `getAge()`、`getMaxAge()`、`getStage()`、`getMoisture()`、`hasBerries()`、`getHoneyLevel()`、`isPersistent()`、`getDistance()`、`getBambooLeaves()`、`getFlowerAmount()` | 小麦、竹子、树叶、蜂巢、耕地 |
| 流体 / 含水 | `isWaterlogged()`、`getLevel()`、`getMaxLevel()`、`getMinLevel()`、`isFalling()`、`isBubbleColumnUp()` / `isBubbleColumnDown()` | 楼梯、台阶、水、岩浆、气泡柱 |
| 形状 / 部件 | `getSlabType()`、`getStairShape()`、`getBedPart()`、`isOccupied()`、`getChestType()`、`getRailShape()`、`getAttachment()`、`getTilt()`、`getThickness()`、`getRotation()`、`get*WallShape()` | 台阶、楼梯、床、大箱子、铁轨、钟、告示牌 |
| 计数类 | `getBites()`、`getCandles()`、`getEggs()`、`getHatched()`、`getLayers()`、`getPickles()`、`getCharges()`、`getDusted()` | 蛋糕、蜡烛、海龟蛋、雪层、重生锚、可疑的沙子 |
| 容器 / 杂项 | `hasBottle0()`~`hasBottle2()`、`hasRecord()`、`hasBook()`、`hasEye()`、`isSlot0Occupied()`~`isSlot5Occupied()`、`isSnowy()`、`isHanging()`、`isSignalFire()`、`isShrieking()`、`getSculkSensorPhase()`、`getTrialSpawnerState()`、`getVaultState()`、`isOminous()` | 酿造台、唱片机、讲台、末地传送门框架、雕纹书架、灯笼、营火、幽匿尖啸体 |

```javascript
// 例：检查面前的门开没开
const block = Player.rayTraceBlock(5, false)
if (block && block.getId().endsWith("_door")) {
    const uni = block.getBlockStateHelper().getUniversal()
    Chat.log(uni.isOpen() ? "门开着" : "门关着")
}
```

!!! warning "只调用该方块真正拥有的属性"
    这些 getter 是给**所有**方块共用的，读取当前方块不存在的属性会直接报错。不确定时先用 `state.toMap()` 看看它到底有哪些属性，再决定调用哪个方法。

!!! tip "属性太多，按分组浏览 d.ts"
    上表只是代表性方法。完整列表在 `JsMacros-2.1.0.d.ts` 的 `UniversalBlockStateHelper` 类中（42705 行起），方法名基本与原版状态属性同名，按上面的分组去搜很快能找到。

## FluidStateHelper：流体状态

从 `BlockStateHelper.getFluidState()` 获得。普通方块也能调用，只是结果为"空流体"。

| 方法 | 返回类型 | 作用 |
| --- | --- | --- |
| `getId()` | `string` | 流体 ID，如 `minecraft:water`、`minecraft:flowing_lava`；非流体为 `minecraft:empty` |
| `isEmpty()` | `boolean` | 是否为空（非流体方块的默认值） |
| `isStill()` | `boolean` | 是否静止（源方块） |
| `getHeight()` | `number` | 流体高度 |
| `getLevel()` | `number` | 流体等级 |
| `hasRandomTicks()` | `boolean` | 是否有随机刻逻辑（岩浆引燃用） |
| `getVelocity(pos)` | `Pos3D` | 该位置对实体施加的推力向量 |
| `getBlockState()` | `BlockStateHelper` | 该流体对应的方块状态 |
| `getBlastResistance()` | `number` | 爆炸抗性 |

```javascript
const fluid = World.getBlock(0, 62, 0).getBlockStateHelper().getFluidState()
if (!fluid.isEmpty()) {
    Chat.log(`${fluid.getId()} 静止=${fluid.isStill()}`)
}
```

## DirectionHelper：方向

来源：`HitResultHelper$Block.getSide()`（命中面）、`UniversalBlockStateHelper.getFacing()` 等（方块朝向）、`Player.getPlayer().getFacingDirection()`（玩家朝向）。

| 方法 | 返回类型 | 作用 |
| --- | --- | --- |
| `getName()` | `string` | 方向名：`down` / `up` / `north` / `south` / `west` / `east` |
| `getAxis()` | `string` | 所在轴：`x` / `y` / `z` |
| `isVertical()` / `isHorizontal()` | `boolean` | 是否竖直 / 水平方向 |
| `isTowardsPositive()` | `boolean` | 是否指向坐标轴正方向 |
| `getYaw()` / `getPitch()` | `number` | 对应的偏航角 / 俯仰角 |
| `getOpposite()` | `DirectionHelper` | 相反方向 |
| `getLeft()` / `getRight()` | `DirectionHelper` | 左转 / 右转 90 度的方向 |
| `getVector()` | `Pos3D` | 单位方向向量，如 north → (0, 0, -1) |
| `pointsTo(yaw)` | `boolean` | 给定偏航角是否最接近该方向 |

`getName()` 的返回值可以直接喂给 `BlockPosHelper.offset(direction)`，两者无缝配合。

## ChunkHelper：区块

来源：`World.getChunk(chunkX, chunkZ)`（注意参数是**区块坐标**，即方块坐标除以 16 取整）或 `Player.getPlayer().getChunk()`。

| 方法 | 返回类型 | 作用 |
| --- | --- | --- |
| `getChunkX()` / `getChunkZ()` | `number` | 区块坐标（不是世界坐标） |
| `getStartingBlock()` | `BlockPosHelper` | 区块起始角（本区块内 x、z 最小处） |
| `getOffsetBlock(xOff, y, zOff)` | `BlockPosHelper` | 相对起始角偏移后的坐标，y 为绝对值 |
| `getMaxBuildHeight()` / `getMinBuildHeight()` | `number` | 最高 / 最低建筑高度 |
| `getHeight()` | `number` | 区块总高度 |
| `getTopYAt(xOff, zOff, heightmap)` | `number` | 某柱最高方块的 y，heightmap 传下面四个原始高度图之一 |
| `getBiome(xOff, y, zOff)` | `string` | 该位置的生物群系 ID |
| `getInhabitedTime()` | `number` | 玩家在此区块累计停留时间（影响局部难度） |
| `getEntities()` | `JavaList<EntityHelper>` | 区块内所有实体 |
| `getTileEntities()` | `JavaList<BlockPosHelper>` | 区块内所有方块实体的坐标 |
| `forEach(includeAir, callback)` | `ChunkHelper` | 遍历区块内每个方块，回调收到 `BlockDataHelper` |
| `containsAny(...ids)` | `boolean` | 区块内是否含**任一**指定方块 |
| `containsAll(...ids)` | `boolean` | 区块内是否含**全部**指定方块 |
| `getHeightmaps()` | 原始集合 | 所有高度图 |
| `getSurfaceHeightmap()` / `getOceanFloorHeightmap()` / `getMotionBlockingHeightmap()` / `getMotionBlockingNoLeavesHeightmap()` | 原始对象 | 四种原始高度图，配合 `getTopYAt` 使用 |

```javascript
// 当前区块里有没有钻石矿？
const player = Player.getPlayer()
const chunk = World.getChunk(Math.floor(player.getX() / 16), Math.floor(player.getZ() / 16))
if (chunk) {
    Chat.log(chunk.containsAny("minecraft:diamond_ore", "minecraft:deepslate_diamond_ore")
        ? "§b本区块有钻石！" : "本区块没有钻石")
}
```

!!! warning "forEach 开销不小"
    一个区块有几万个方块位置，`forEach(false, ...)`（跳过空气）会比 `forEach(true, ...)` 快很多。大范围搜方块请优先用[世界扫描器](world_scanner.md)。

## HitResultHelper：射线命中结果

`Player.rayTraceBlock(distance, fluid)` 直接返回 `BlockDataHelper`，够用就用它；需要知道**命中的是哪个面**时用 `Player.detailedRayTraceBlock(distance, fluid)`，它返回 `HitResultHelper$Block`。交互管理器的 `getTarget()` 也返回 `HitResultHelper`（可能是方块或实体）。

| 方法 | 返回类型 | 说明 |
| --- | --- | --- |
| `getPos()` | `Pos3D` | 命中点精确坐标（基类方法） |
| `asBlock()` / `asEntity()` | 子类或 `null` | 转成方块 / 实体命中结果（基类方法） |
| `Block.getBlockPos()` | `BlockPosHelper \| null` | 命中的方块坐标 |
| `Block.getSide()` | `DirectionHelper \| null` | 命中的面 |
| `Block.isMissed()` | `boolean` | 是否没打中 |
| `Block.isInsideBlock()` | `boolean` | 起点是否在方块内部 |
| `Entity.getEntity()` | `EntityHelper` | 命中的实体 |

```javascript
const hit = Player.detailedRayTraceBlock(5, false)
if (hit && !hit.isMissed()) {
    const pos = hit.getBlockPos()
    Chat.log(`打中 ${pos.getX()}, ${pos.getY()}, ${pos.getZ()} 的 ${hit.getSide().getName()} 面`)
}
```

## 实战示例

### 1. 检测面前的方块能不能挖

综合硬度、是否需要工具、手上物品是否合适：

```javascript
const block = Player.rayTraceBlock(4.5, false)
if (!block) {
    Chat.log("准星前方 4.5 格内没有方块")
} else {
    const state = block.getBlockStateHelper()
    const name = block.getName().getString()
    const hardness = state.getHardness()

    if (hardness < 0) {
        Chat.log(`§c${name} 无法被挖掘（如基岩）`)
    } else {
        const hand = Player.getPlayer().getMainHand()
        Chat.log(`方块: ${name}  硬度: ${hardness}  必须用工具: ${state.isToolRequired()}`)
        if (hand.isSuitableFor(state)) {
            Chat.log("§a当前手持物品挖掘后会正常掉落")
        } else if (state.isToolRequired()) {
            Chat.log("§c用当前物品挖掘不会掉落，换个工具吧")
        } else {
            Chat.log("§e空手也能挖，只是可能比较慢")
        }
    }
}
```

### 2. 扫描下方岩浆预警

挖矿往下打洞前跑一下，或挂在 `Tick` 事件里常驻：

```javascript
const player = Player.getPlayer()
if (player) {
    const feet = player.getBlockPos()
    for (let i = 1; i <= 20; i++) {
        const block = World.getBlock(feet.down(i))
        if (!block) break                        // 区块未加载
        const state = block.getBlockStateHelper()
        if (state.isAir()) continue              // 空气，继续往下看
        if (state.getFluidState().getId().includes("lava")) {
            Chat.actionbar(`§c警告：正下方 ${i} 格是岩浆！`)
        }
        break                                    // 碰到第一个非空气方块就停
    }
}
```

### 3. 读告示牌文字（NBT 实战）

告示牌是"有方块实体"的典型例子，文字存在 NBT 的 `front_text.messages`（背面是 `back_text.messages`）里，每行是一段 JSON 文本：

```javascript
const block = Player.rayTraceBlock(5, false)
if (block && block.getId().endsWith("sign")) {   // 普通、墙上、悬挂告示牌 ID 都以 sign 结尾
    const nbt = block.getNBT()
    if (nbt && nbt.has("front_text")) {
        const front = nbt.get("front_text")
        if (front.isCompound()) {
            const messages = front.asCompoundHelper().get("messages")
            if (messages && messages.isList()) {
                const list = messages.asListHelper()
                for (let i = 0; i < list.length(); i++) {
                    const json = list.get(i).asString()             // 形如 '{"text":"你好"}'
                    const text = Chat.createTextHelperFromJSON(json)
                    if (text) Chat.log(`第 ${i + 1} 行: ${text.getString()}`)
                }
            }
        }
    }
} else {
    Chat.log("请把准星对准一块告示牌再运行")
}
```

### 4. 找脚下附近的箱子

```javascript
const player = Player.getPlayer()
if (player) {
    const center = player.getBlockPos()
    World.iterateSphere(center, 6, true, JavaWrapper.methodToJava((block) => {
        if (block.getId() === "minecraft:chest") {
            Chat.log(`箱子: ${block.getX()} ${block.getY()} ${block.getZ()}`)
        }
    }))
}
```

!!! tip "更大范围请用世界扫描器"
    `iterateSphere` / `iterateBox` 适合小范围；要在几十个区块里找方块，用 `World.findBlocksMatching(...)` 或性能更好的[世界扫描器](world_scanner.md)。

## 相关页面

- [世界](world.md)——`World.getBlock`、`iterateSphere` 等入口方法的完整说明
- [世界扫描器](world_scanner.md)——大范围高性能方块搜索
- [位置与向量](position.md)——`Pos3D`、`Vec3D` 与坐标运算
- [HUD 渲染](hud.md)——把找到的方块画到屏幕上（ESP、路径点）
