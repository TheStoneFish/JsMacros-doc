---
icon: lucide/globe-2
---

# 世界与方块

`World` 用来读取当前世界、方块、区块、实体、维度、时间、天气和声音。多数方法在世界未加载时会返回 `null`，先判空是好习惯。

## 世界是否加载

```javascript
if (!World.isWorldLoaded()) {
    Chat.log("还没进入世界")
    return
}
```

## 读取方块

```javascript
const block = World.getBlock(0, 64, 0)
if (block) {
    Chat.log(`${block.getId()} ${block.getName().getString()}`)
}
```

`getBlock` 也能接 `Pos3D` 或 `BlockPosHelper`。

常用返回值是 `BlockDataHelper`，继续看 [方块 Helper](blocks.md)。

## 搜索方块

```javascript
const blocks = World.findBlocksMatching("minecraft:diamond_ore", 2)
if (blocks) {
    Chat.log(`附近区块找到 ${blocks.size()} 个钻石矿`)
}
```

重载包括：

| 写法 | 含义 |
| --- | --- |
| `findBlocksMatching(id, chunkrange)` | 以玩家所在区块为中心 |
| `findBlocksMatching(ids, chunkrange)` | 多个方块 ID |
| `findBlocksMatching(centerX, centerZ, id, chunkrange)` | 指定区块中心 |
| `findBlocksMatching(blockFilter, stateFilter, chunkrange)` | 用回调过滤 |

!!! note "chunkrange 不是格数"
    `chunkrange=1` 表示玩家所在区块加周围一圈区块。范围很大时会卡，请从小范围开始。

## 遍历区域

```javascript
const center = Player.getPlayer().getBlockPos()
World.iterateSphere(center, 4, true, JavaWrapper.methodToJava((block) => {
    if (block.getId() === "minecraft:chest") {
        Chat.log(`箱子: ${block.getX()}, ${block.getY()}, ${block.getZ()}`)
    }
}))
```

`iterateBox(pos1, pos2, ignoreAir, callback)` 适合扫描长方体区域。

## 实体

```javascript
const zombies = World.getEntities(8, "zombie")
if (zombies && zombies.size() > 0) {
    Chat.log(zombies.get(0).getName().getString())
}
```

常用重载：

| 方法 | 作用 |
| --- | --- |
| `getEntities()` | 所有已加载实体 |
| `getEntities(distance)` | 指定距离内实体 |
| `getEntities(distance, ...types)` | 指定距离和类型 |
| `getEntities(filter)` | 用回调过滤 |

更多实体方法看 [实体](entities.md)。

## 玩家列表

```javascript
const players = World.getLoadedPlayers()
if (players) {
    for (const p of players) {
        Chat.log(`${p.getPlayerName()} ${p.getHealth()}`)
    }
}
```

服务器 Tab 列表：

```javascript
const entry = World.getPlayerEntry("Steve")
if (entry) {
    Chat.log(`${entry.getName()} ping=${entry.getPing()}`)
}
```

## 射线检测

```javascript
const player = Player.getPlayer()
if (player) {
    const eye = player.getEyePos()
    const look = Player.createLookingVector(player).scale(5)
    const end = eye.add(look.getDeltaX(), look.getDeltaY(), look.getDeltaZ())
    const block = World.rayTraceBlock(eye.getX(), eye.getY(), eye.getZ(), end.getX(), end.getY(), end.getZ(), false)
    if (block) Chat.log(block.getId())
}
```

更常用的准心检测可以直接用 `Player.rayTraceBlock(distance, fluid)`。

## 维度、时间和天气

```javascript
Chat.log(World.getDimension())
Chat.log(World.getBiome())
Chat.log(World.getTime())
Chat.log(World.getTimeOfDay())
Chat.log(World.isDay())
Chat.log(World.isRaining())
```

| 方法 | 作用 |
| --- | --- |
| `getWorldIdentifier()` | 世界标识 |
| `getRespawnPos()` | 重生点 |
| `getDifficulty()` | 难度 |
| `getMoonPhase()` | 月相 |
| `getSkyLight(x, y, z)` | 天空光照 |
| `getBlockLight(x, y, z)` | 方块光照 |

## 声音

```javascript
World.playSound("minecraft:block.note_block.pling", 1, 1)
```

也可以指定坐标：

```javascript
const p = Player.getPlayer().getPos()
World.playSound("minecraft:block.note_block.pling", 1, 1, p.getX(), p.getY(), p.getZ())
```
