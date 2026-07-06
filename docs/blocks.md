---
icon: lucide/box
---

# 方块 Helper

世界 API 里最常见的方块相关类型是 `BlockDataHelper`、`BlockHelper`、`BlockStateHelper` 和 `BlockPosHelper`。

## 这几个类型怎么分？

| 类型 | 表示 | 例子 |
| --- | --- | --- |
| `BlockDataHelper` | 某个坐标处的方块数据 | `World.getBlock(x, y, z)` |
| `BlockHelper` | 方块种类本身 | 石头、草方块、箱子 |
| `BlockStateHelper` | 方块状态 | 朝向、水含量、是否点亮 |
| `BlockPosHelper` | 整数方块坐标 | `player.getBlockPos()` |

## BlockDataHelper

```javascript
const block = World.getBlock(0, 64, 0)
if (block) {
    Chat.log(block.getId())
    Chat.log(block.getName().getString())
    Chat.log(`${block.getX()}, ${block.getY()}, ${block.getZ()}`)
}
```

常用方法：

| 方法 | 作用 |
| --- | --- |
| `getId()` | 方块 ID |
| `getName()` | 方块显示名，返回 `TextHelper` |
| `getX()` / `getY()` / `getZ()` | 坐标 |
| `getNBT()` | 方块实体 NBT，可能为 `null` |
| `getBlockState()` | 状态键值映射 |
| `getBlockStateHelper()` | 状态 helper |
| `getBlock()` | 方块种类 helper |
| `getBlockPos()` | 坐标 helper |
| `getRawBlock()` / `getRawBlockState()` | Minecraft 原始对象 |

!!! tip "别混 ID 和显示名"
    `getId()` 才是脚本判断用的 `minecraft:stone`；`getName()` 是能显示给人的文本。

## BlockHelper

`BlockHelper` 描述方块种类，不绑定具体坐标。

| 方法 | 作用 |
| --- | --- |
| `getDefaultState()` | 默认方块状态 |
| `getDefaultItemStack()` | 对应物品 |
| `getHardness()` | 硬度 |
| `getBlastResistance()` | 爆炸抗性 |
| `getSlipperiness()` | 滑度 |
| `getJumpVelocityMultiplier()` | 跳跃速度倍率 |
| `getVelocityMultiplier()` | 移动速度倍率 |
| `getTags()` | 方块标签 |
| `getStates()` | 可能状态 |
| `canMobSpawnInside()` | 生物是否能在内部生成 |

```javascript
const helper = World.getBlock(0, 64, 0).getBlock()
Chat.log(`hardness=${helper.getHardness()}`)
```

## BlockStateHelper

```javascript
const state = World.getBlock(0, 64, 0).getBlockStateHelper()
Chat.log(state.isAir())
Chat.log(state.isReplaceable())
```

常用方法：

| 方法 | 作用 |
| --- | --- |
| `getId()` | 状态对应方块 ID |
| `getFluidState()` | 流体状态 |
| `getHardness()` | 当前状态硬度 |
| `getLuminance()` | 亮度 |
| `isAir()` | 是否空气 |
| `isLiquid()` | 是否液体 |
| `isSolid()` | 是否实体方块 |
| `isReplaceable()` | 是否可被替换 |
| `hasBlockEntity()` | 是否有方块实体 |
| `hasComparatorOutput()` | 是否有比较器输出 |
| `allowsSpawning(pos, entity)` | 是否允许指定实体生成 |

## BlockPosHelper

```javascript
const pos = Player.getPlayer().getBlockPos()
const below = pos.down()
const twoNorth = pos.north(2)
```

| 方法 | 作用 |
| --- | --- |
| `up()` / `down()` | 上下偏移 1 格 |
| `north()` / `south()` / `east()` / `west()` | 水平偏移 1 格 |
| `up(distance)` 等 | 按距离偏移 |
| `offset(direction)` | 按方向字符串偏移 |
| `offset(x, y, z)` | 按坐标偏移 |
| `distanceTo(...)` | 到实体、坐标或另一个位置的距离 |
| `toNetherCoords()` / `toOverworldCoords()` | 下界/主世界坐标换算 |
| `toPos3D()` | 转小数坐标 |

## 示例：找脚下附近的箱子

```javascript
const player = Player.getPlayer()
if (!player) return

const center = player.getBlockPos()
World.iterateSphere(center, 6, true, JavaWrapper.methodToJava((block) => {
    if (block.getId() === "minecraft:chest") {
        Chat.log(`箱子: ${block.getX()} ${block.getY()} ${block.getZ()}`)
    }
}))
```

