---
icon: lucide/axis-3d
---

# 位置与向量

JsMacros 里常见的位置类型有 `Pos2D`、`Pos3D`、`Vec2D`、`Vec3D` 和 `BlockPosHelper`。

## 创建位置

```javascript
const p3 = Player.createPos(1, 64, 2)
const p2 = Player.createPos(1, 2)
const block = Player.createBlockPos(1, 64, 2)
```

`Pos3D` 是小数坐标；`BlockPosHelper` 是方块整数坐标。

## 创建向量

```javascript
const vec3 = Player.createVec(0, 64, 0, 10, 70, 10)
const look = Player.createLookingVector(Player.getPlayer())
const look2 = Player.createLookingVector(90, 0)
```

`createVec(x1, y1, z1, x2, y2, z2)` 表示从点 1 到点 2 的三维向量。

## Pos2D / Pos3D

常用运算：

```javascript
const pos = Player.createPos(1, 64, 2)
const moved = pos.add(0, 1, 0)
const scaled = pos.multiply(2, 2, 2)
```

常见方法：

| 方法 | 作用 |
| --- | --- |
| `add(...)` | 加位置/坐标 |
| `sub(...)` | 减位置/坐标 |
| `multiply(...)` | 乘位置/坐标 |
| `divide(...)` | 除位置/坐标 |
| `scale(scale)` | 整体缩放 |
| `distanceTo(...)` | 距离 |
| `toString()` | 字符串 |

## Vec2D / Vec3D

向量适合表示方向和位移。

```javascript
const player = Player.getPlayer()
const look = Player.createLookingVector(player)
Chat.log(look.getMagnitude())
```

常见方法包括加减乘除、长度、点积、归一化等。具体签名以 `JsMacros-2.1.0.d.ts` 的 `Vec2D` / `Vec3D` 为准。

## BlockPosHelper

```javascript
const pos = Player.getPlayer().getBlockPos()
const target = pos.down().north(2)

Chat.log(target.distanceTo(Player.getPlayer()))
```

`BlockPosHelper` 更适合世界方块操作，详见 [方块 Helper](blocks.md)。

## 方向字符串

```ts
type Direction = "up" | "down" | "north" | "south" | "east" | "west"
```

方块交互常用：

```javascript
Player.interactions().interactBlock(pos.getX(), pos.getY(), pos.getZ(), "up", false)
```
