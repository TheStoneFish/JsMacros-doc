---
icon: lucide/users
---

# 实体

`EntityHelper` 是所有实体 helper 的基类。玩家、动物、怪物、掉落物等会有更具体的子类，但常用读取都在基类和 `LivingEntityHelper` 里。

## 获取实体

```javascript
const entities = World.getEntities(6)
if (!entities) return

for (const entity of entities) {
    Chat.log(`${entity.getType()} ${entity.distanceTo(Player.getPlayer())}`)
}
```

按类型过滤：

```javascript
const zombies = World.getEntities(8, "zombie")
```

## 基础信息

| 方法 | 作用 |
| --- | --- |
| `getType()` | 实体 ID |
| `getName()` | 显示名，返回 `TextHelper` |
| `getUUID()` | UUID |
| `getPos()` / `getBlockPos()` / `getEyePos()` | 坐标 |
| `getYaw()` / `getPitch()` | 朝向 |
| `getVelocity()` | 速度 |
| `getChunkPos()` / `getChunk()` | 所在区块 |
| `getBiome()` | 所在生物群系 |
| `getNBT()` | NBT |

```javascript
const e = World.getEntities(4).get(0)
Chat.log(`${e.getName().getString()} ${e.getX()} ${e.getY()} ${e.getZ()}`)
```

## 状态判断

```javascript
if (entity.isAlive() && entity.isReallyAlive()) {
    Chat.log("实体还活着")
}
```

| 方法 | 作用 |
| --- | --- |
| `isAlive()` | 实体是否存活 |
| `isReallyAlive()` | 2.1.0 推荐，更适合过滤异常/卸载状态 |
| `isGlowing()` | 是否发光 |
| `isInLava()` | 是否在岩浆 |
| `isOnFire()` | 是否着火 |
| `isSneaking()` / `isSprinting()` | 潜行/疾跑 |

## 距离和视线

```javascript
const player = Player.getPlayer()
const nearest = World.getEntities(8, "zombie").get(0)

Chat.log(nearest.distanceTo(player))
Chat.log(player.canSeeEntity(nearest))
```

实体可对方块/实体做射线：

```javascript
const block = player.rayTraceBlock(5, false)
const target = player.rayTraceEntity(5)
```

## LivingEntityHelper

生物、玩家、怪物通常能转成 `LivingEntityHelper`。

```javascript
const living = entity.asLiving()
Chat.log(`${living.getHealth()} / ${living.getMaxHealth()}`)
```

常用方法：

| 方法 | 作用 |
| --- | --- |
| `getHealth()` / `getMaxHealth()` | 血量 |
| `getArmor()` | 护甲值 |
| `getStatusEffects()` | 状态效果 |
| `hasStatusEffect(id)` | 是否有某状态 |
| `getMainHand()` / `getOffHand()` | 主副手物品 |
| `getHeadArmor()` 等 | 装备 |
| `isOnGround()` | 是否在地面 |
| `isFallFlying()` | 是否鞘翅飞行 |
| `canSeeEntity(entity)` | 是否可见目标 |
| `isReallyAliveAndHealthy()` | 2.1.0 推荐健康判断 |

## PlayerEntityHelper

玩家实体额外提供：

| 方法 | 作用 |
| --- | --- |
| `getPlayerName()` | 玩家名字符串 |
| `getXP()` / `getXPLevel()` | 经验 |
| `getAttackCooldownProgress()` | 攻击冷却 |
| `getAbilities()` | 飞行、创造、速度等能力 |
| `getFishingBobber()` | 鱼漂实体 |

```javascript
const players = World.getLoadedPlayers()
if (players) {
    for (const p of players) {
        Chat.log(`${p.getPlayerName()} Lv.${p.getXPLevel()}`)
    }
}
```

## 发光标记

```javascript
entity.setGlowing(true)
entity.setGlowingColor(0x00FF00)
Client.waitTick(100)
entity.resetGlowingColor()
entity.resetGlowing()
```

!!! warning "客户端视觉效果"
    发光设置主要影响本地显示。多人服务器里不要把它当作服务端状态。

## 类型转换

| 方法 | 目标 |
| --- | --- |
| `asClientPlayer()` | 本地玩家 |
| `asPlayer()` | 玩家 |
| `asLiving()` | 生物 |
| `asAnimal()` | 动物 |
| `asItem()` | 掉落物 |
| `asVillager()` | 村民 |
| `asMerchant()` | 可交易实体 |
| `asServerEntity()` | 服务端实体 helper，可能为 `null` |

