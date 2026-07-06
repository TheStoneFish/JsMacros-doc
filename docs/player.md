---
icon: lucide/angry
---

# 玩家

!!! note "2.x 推荐"
    读取玩家信息用 `Player.getPlayer()`；攻击、交互、挖掘等动作优先用 `Player.getInteractionManager()` 或 `Player.interactions()`。旧的 `player.attack()`、`player.interactBlock()` 等方法还能在类型里看到，但已迁移到交互管理器。

## 获取玩家对象

```javascript
const player = Player.getPlayer()
```

获取到玩家对象后, 后面的操作都需要通过这个对象进行。

`Player.getPlayer()` 可能返回 `null`，比如还没进入世界时：

```javascript
const player = Player.getPlayer()
if (!player) {
    Chat.log("请先进入世界")
    return
}
```

## 获取玩家名字

```javascript
const player = Player.getPlayer()
const playerName = player.getName().getString()
```

在JsMacros里面获取到的文本内容都是经过`TextHelper`包装的, 可以使用JsMacros的方式来处理文本内容, 如果需要获取字符串的话需要再进行`getString()`。
!!! question "输出玩家名字"
    接下来尝试在屏幕输出玩家名字。

    需要方法:
    > `Player.getPlayer()` 方法可以获取到玩家对象。

    > `player.getName().getString()` 方法可以获取到玩家名字。

    > `Chat.log(playerName)`方法可以输出文本到屏幕。
    ???- tip "答案"
        ```javascript
        const player = Player.getPlayer()
        const playerName = player.getName().getString()
        Chat.log(playerName)
        ```
        Chat.log(playerName)方法可以输出文本到屏幕。

## 攻击

!!! warning "旧写法说明"
    本节保留 `player.attack()` 的教程写法，方便读懂旧脚本。新脚本建议改用 [交互管理器](interaction.md)：`Player.interactions().attack()` 或 `Player.interactions().attack(entity)`。

### attack()

```javascript
const player = Player.getPlayer()
player.attack()
```

使用`attack()`方法可以对准心处进行攻击, 相当于按下左键。

### attack(entity)

```javascript
const player = Player.getPlayer()
const targetEntity = World.getEntities(4.5, 'zombie')[0]
player.attack(targetEntity)
```

这里通过`World.getEntities('zombie')`获取到一个`EntityHelper`列表, 然后将列表的第一个元素作为目标实体进行攻击。
使用`attack(entity)`方法可以对指定实体进行攻击。
!!! question "杀戮光环"
    制作一个杀戮光环, 让玩家击杀附近的怪物。
    需要方法:
    > `Player.getPlayer()` 方法可以获取到玩家对象。

    > `World.getEntities(4.5, 'zombie')` 方法可以获取到距离玩家4.5格以内的`zombie`实体。

    > `player.attack(entity)` 方法可以对指定实体进行攻击。

    > `Time.sleep(100)` 方法可以让程序暂停100ms。
    
    ???- tip "答案"
        ``` javascript
        while(true){
            const player = Player.getPlayer()
            const targetEntity = World.getEntities(4.5, 'zombie')[0]
            player.attack(targetEntity)
            Time.sleep(100)
        }
        ```
        这里使用了一个死循环, 每隔100ms就对距离玩家4.5格以内的第一个`zombie`实体进行攻击。
        
        注: "第一个"并不是最近的一个, 如果需要获取最近的，需要排序，后面会教

        死循环无法退出，需要去jsmacros模组菜单里点左下角"正在运行"然后点×关掉
        后面会教如何使用按键开关脚本

## 交互

!!! warning "旧写法说明"
    `player.interact()`、`player.interactEntity()`、`player.interactBlock()`、`player.interactItem()` 已迁移到 `Player.getInteractionManager()`。新脚本优先写 `Player.interactions().interactBlock(...)`。

### interact()

```javascript
const player = Player.getPlayer()
player.interact()
```

使用`interact()`方法可以与准心处进行交互, 相当于按下右键。

### interactEntity(entity, offHand)

```javascript
const player = Player.getPlayer()
const targetEntity = World.getEntities(4.5, 'villager')[0]
player.interactEntity(targetEntity, false)
```

使用`interactEntity(entity, offHand)`方法可以与指定实体进行交互。

`offHand`参数是控制交互用的左右手的, 左手为true, 右手为false。
!!! warning "注意反作弊"
    此方法会产生原版不可能发生的发包——即使你交互的村民在你的交互距离内，并且你正看着它，使用此方法交互也会让你被反作弊检测到！

    1. 用右手交互，会被检测 (GrimAC-PacketOrderC) 因为这个方法少发了 InteractAt 包 (1.8+)

    2. 如果直接使用左手交互，则问题更大。在报PacketOrderC的同时还会报PacketOrderD (1.9+)

        ^^GrimAC-PacketOrderD：对于 1.9+ 客户端 如果副手交互发生，通常意味着主手交互已经发生。但这个方法漏发了主手交互包^^

    所以在有反作弊的服请谨慎使用此方法！
    可以替换为转头到目标实体, 然后用`interact()`方法进行交互。(当然 转头要是写不好也会被检测到就是了)

    ==本提示只会出现在这种JsMacros的方法本身有隐藏问题的地方。在其他明显会被反作弊检测到的方法或实例中不会出现==

### interactBlock(x, y, z, direction, offHand)

使用`interactBlock(x, y, z, direction, offHand)`方法可以与指定方块进行交互。

| 参数         | 描述             |
| ----------- | -----------------|
| `x`         | 交互目标方块的x坐标 |
| `y`         | 交互目标方块的y坐标 |
| `z`         | 交互目标方块的z坐标 |
| `direction` | 交互的方向, 可以是 `'down'`, `'up'`, `'north'`, `'south'`, `'west'`, `'east'`|
| `offHand` | 交互用的左右手, 左手为true, 右手为false |

```javascript
const player = Player.getPlayer()
player.interactBlock(0, -61, 0, 'up', false)
```

!!! note "实验"
    上面的代码可以让你和坐标为`(0, -61, 0)`的方块进行交互, 交互方向为顶面 `(up)`

    你可以新建一个超平坦世界，然后`/tp 0 -60 0`再往边上走一格并手持任意方块 来在游戏中测试这两行代码
    
    它的效果等于你将准心对准位于`(0, -61, 0)`的方块的顶面, 然后按下右键

但它不止能做到这样，它可以在空中放方块！
现在继续呆在这个坐标，将上面函数里的`0, -61, 0`改为`0, -57, 0`再运行试试看吧！

哇，他居然在空中放了一个方块，把`(0, -57, 0)`的空气变成了你手上拿的方块！

??? question "Why?"
    好奇怪欸，刚刚我们与`y=-61`的草方块顶面交互，他在`y=-60`处放了方块

    为什么现在我们与`y=-57`的空气交互，他就在原位`y=-57`处放了方块？

    好问题！这是因为空气是可被替换的方块！为便于你理解，你可以现在在草地上撒骨粉。草也是可被替换的方块，你现在拿着方块对准刚刚长出的草，按右键
    !!! success "wow!"
        wow！草的位置直接被替换为了手上的方块！

        没错。空气也是如此。现在知道为啥我们与`y=-57`的空气交互，他就在原位`y=-57`处放了方块了吧！

    你可以再次验证：你现在不改脚本，回到刚刚放的方块边上，再运行一次脚本。

    脚本会和`(0, -57, 0)`的你刚刚放的方块的顶面交互

    然后你手上的方块就会被放到`(0, -56, 0)`的位置！

    在此间你根本没有改变脚本内容。但脚本的两次执行却带来了不同的结果。

    在以后的写脚本旅途中你一定会遇到类似这样的情况。**理解这样的问题本质，很有利于你以后的debug！**

!!! question "种树光环"
    制作一个种树光环, 让玩家自动与旁边的草方块交互进行种树。
    需要方法:
    > `Player.getPlayer()` 方法可以获取到玩家对象。

    > `World.findBlocksMatching("minecraft:grass_block", 1)` 方法可以获取到玩家所在区块以及周围一圈区块内的所有草方块。

    > `Player.getPlayer().getBlockPos().distanceTo(block)` 方法可以获取到方块与玩家的距离。

    > `player.interactBlock(x, y, z, direction, offHand)` 方法可以与指定方块进行交互。

    > `Time.sleep(100)` 方法可以让程序暂停100ms。
    
    ???- tip "答案"
        ``` javascript
        while (true) {
            const player = Player.getPlayer()
            const targetBlocks = World.findBlocksMatching("minecraft:grass_block", 1)
            for (let block of targetBlocks) { // 遍历草方块列表里所有草方块
                // 判断每个草方块距离玩家的距离，若在玩家周围4.5格范围内则用主手与它的顶面交互
                if (Player.getPlayer().getBlockPos().distanceTo(block) < 4.5) {
                    player.interactBlock(block.getX(), block.getY(), block.getZ(), 'up', false)
                }
            }
            Time.sleep(100)
        }
        ```
        这里使用了一个死循环, 每隔100ms就搜索玩家所在区块及周围一圈区块里所有草方块，遍历每个方块，判断是否在玩家周围4.5格范围内，若在 则用主手与它的顶面交互。

        现在拿上树苗试试吧！

### interactItem(offHand)

使用`interactItem(offHand)`方法可以用你手中的物品进行交互。如用水瓶接水或打开服务器菜单等，大部分情况等效于`interact()`，但这个interactItem(offHand)可以控制左右手

`offHand`参数是控制交互用的左右手的, 左手为true, 右手为false。

```javascript
const player = Player.getPlayer()
player.interactItem(false)
```

## Player 全局库速查

| 方法 | 作用 |
| --- | --- |
| `openInventory()` | 获取当前背包/容器对象 |
| `getPlayer()` | 获取本地玩家实体，可能为 `null` |
| `getInteractionManager()` / `interactions()` | 获取交互管理器 |
| `getGameMode()` / `setGameMode(gameMode)` | 读取/设置游戏模式 |
| `rayTraceBlock(distance, fluid)` | 准心方块射线 |
| `detailedRayTraceBlock(distance, fluid)` | 更详细的方块命中结果 |
| `rayTraceEntity(distance)` | 准心实体射线 |
| `writeSign(...)` | 写告示牌 |
| `takeScreenshot(...)` | 截图 |
| `takePanorama(...)` | 全景截图 |
| `getStatistics()` | 统计信息 |
| `getReach()` | 触及距离 |
| `createPlayerInput(...)` | 创建移动输入 |
| `addInput(input)` / `addInputs(inputs)` | 注入移动输入 |
| `clearInputs()` | 清空输入队列 |
| `predictInput(input)` / `predictInputs(inputs)` | 预测移动结果 |
| `createPos(...)` / `createBlockPos(...)` | 创建位置 |
| `createVec(...)` / `createLookingVector(...)` | 创建向量 |

## 本地玩家常用信息

```javascript
const player = Player.getPlayer()
if (!player) return

Chat.log(player.getPlayerName())
Chat.log(`${player.getX()}, ${player.getY()}, ${player.getZ()}`)
Chat.log(`血量: ${player.getHealth()} / ${player.getMaxHealth()}`)
Chat.log(`经验等级: ${player.getXPLevel()}`)
```

常用方法来自 `EntityHelper`、`LivingEntityHelper` 和 `PlayerEntityHelper`：

| 方法 | 作用 |
| --- | --- |
| `getPos()` / `getBlockPos()` / `getEyePos()` | 坐标 |
| `getYaw()` / `getPitch()` | 朝向 |
| `getVelocity()` | 速度 |
| `getHealth()` / `getMaxHealth()` | 血量 |
| `getMainHand()` / `getOffHand()` | 主副手物品 |
| `getStatusEffects()` | 状态效果 |
| `getAttackCooldownProgress()` | 攻击冷却 |
| `getAbilities()` | 飞行、创造等能力 |

## 准心检测

```javascript
const block = Player.rayTraceBlock(5, false)
if (block) {
    Chat.log(`看着: ${block.getId()}`)
}

const entity = Player.rayTraceEntity(5)
if (entity) {
    Chat.log(`看着实体: ${entity.getType()}`)
}
```

`fluid=true` 时方块射线会考虑流体。

## 移动输入

移动输入适合做短暂、可预测的移动，不适合无脑高速循环。

```javascript
const input = Player.createPlayerInput(1, 0, Player.getPlayer().getYaw(), 0, false, false, true)
Player.addInput(input)
```

清空：

```javascript
Player.clearInputs()
```

预测：

```javascript
const predicted = Player.predictInput(input)
Chat.log(predicted)
```
