---
icon: lucide/mouse-pointer-click
---

# 交互管理器

2.x 推荐用 `Player.getInteractionManager()` 或别名 `Player.interactions()` 做攻击、交互、挖掘和目标覆盖。旧的 `player.attack()`、`player.interactBlock()` 仍能见到，但类型文件里已经标注迁移到这里。

## 获取对象

```javascript
const im = Player.getInteractionManager()
if (!im) {
    Chat.log("当前没有交互管理器")
    return
}
```

`Player.interactions()` 是同义入口：

```javascript
const im = Player.interactions()
```

## 目标覆盖

```javascript
const pos = Player.getPlayer().getBlockPos().down()
im.setTarget(pos, "up")
im.interact()
im.clearTargetOverride()
```

常用方法：

| 方法 | 作用 |
| --- | --- |
| `setTarget(x, y, z)` | 覆盖目标方块 |
| `setTarget(x, y, z, direction)` | 覆盖目标方块和面 |
| `setTarget(pos, direction)` | 用 `BlockPosHelper` |
| `setTarget(entity)` | 覆盖目标实体 |
| `getTarget()` | 当前目标 |
| `getTargetedBlock()` | 当前目标方块 |
| `getTargetedEntity()` | 当前目标实体 |
| `clearTargetOverride()` | 清除覆盖 |
| `hasTargetOverride()` | 是否有覆盖 |

## 攻击

```javascript
const target = World.getEntities(4.5, "zombie")?.get(0)
if (target) {
    Player.interactions().attack(target)
}
```

攻击准心目标：

```javascript
Player.interactions().attack()
```

攻击方块某个面：

```javascript
Player.interactions().attack(0, 64, 0, "up")
```

部分方法有 `await` 重载，用来等待动作完成。

## 交互

```javascript
Player.interactions().interact()
Player.interactions().interactItem(false)
Player.interactions().interactBlock(0, 64, 0, "up", false)
```

| 方法 | 作用 |
| --- | --- |
| `interact()` | 对当前目标右键 |
| `interactEntity(entity, offHand)` | 右键实体 |
| `interactItem(offHand)` | 使用手中物品 |
| `interactBlock(x, y, z, direction, offHand)` | 右键方块某一面 |
| `holdInteract(holding)` | 持续按住/松开使用 |
| `holdInteract(ticks)` | 持续使用指定 tick |

`offHand=false` 表示主手，`true` 表示副手。

## 挖掘

挖掘方块：

```javascript
const result = Player.interactions().breakBlock(Player.getPlayer().getBlockPos().down())
if (result) {
    Chat.log(result.getReason())
}
```

异步挖掘：

```javascript
Player.interactions().breakBlockAsync(JavaWrapper.methodToJava((result) => {
    Chat.log(result)
}))
```

检查是否正在挖：

```javascript
if (Player.interactions().isBreakingBlock()) {
    Chat.actionbar("正在挖方块")
}
```

## 目标检查

```javascript
im.setTargetRangeCheck(true, true)
im.setTargetAirCheck(true, true)
im.setTargetShapeCheck(true, true)
```

| 方法 | 作用 |
| --- | --- |
| `setTargetRangeCheck(enabled, autoClear)` | 距离检查 |
| `setTargetAirCheck(enabled, autoClear)` | 空气检查 |
| `setTargetShapeCheck(enabled, autoClear)` | 形状检查 |
| `resetTargetChecks()` | 重置检查 |

!!! warning "反作弊提醒"
    目标覆盖和直接交互可能产生原版客户端不容易做到的包序列。多人服务器里优先使用准心对准后的普通交互，避免跨距离、隔墙或过高频操作。
