---
icon: lucide/angry
---

# 玩家

`Player` 是 JsMacros 的全局库之一，负责一切"与你自己这个玩家有关"的操作：获取玩家对象、读写游戏模式、准心射线检测、写告示牌、截图、统计信息，以及往游戏里注入移动输入。

!!! note "2.x 推荐"
    读取玩家信息用 `Player.getPlayer()`；攻击、交互、挖掘等动作优先用 `Player.getInteractionManager()` 或 `Player.interactions()`。旧的 `player.attack()`、`player.interactBlock()` 等方法还能在类型里看到，但已迁移到[交互管理器](interaction.md)。

本页内容：

- 新手教程：获取玩家对象、攻击、交互（保留旧写法讲解，方便读懂旧脚本）
- `Player` 全局库全部函数参考（分组表）
- `ClientPlayerEntityHelper` 完整参考（`getPlayer()` 返回的类型）
- `PlayerEntityHelper`、`PlayerAbilitiesHelper` 方法表
- 移动输入（`PlayerInput`）教学与 `StatsHelper` 统计信息

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

!!! tip "更直接的方式"
    `player.getPlayerName()` 直接返回字符串形式的玩家真实名字（不是显示名），不需要再 `getString()`。

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

## Player 全局库完整参考

以下方法全部直接挂在全局对象 `Player` 上，随时可用，无需先 `getPlayer()`。

### 核心入口

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getPlayer()` | `ClientPlayerEntityHelper` 或 `null` | 获取本地玩家实体包装，未进世界时为 `null` |
| `openInventory()` | `Inventory` | 获取当前背包/容器操作对象，见[背包](inventory.md) |
| `getInteractionManager()` | `InteractionManagerHelper` 或 `null` | 获取交互管理器，见[交互管理器](interaction.md) |
| `interactions()` | 同上 | `getInteractionManager()` 的别名，写起来更短 |

### 游戏模式与触及距离

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getGameMode()` | 字符串 | 当前游戏模式 |
| `setGameMode(gameMode)` | 无 | 设置游戏模式，可用值 `"survival"` / `"creative"` / `"adventure"` / `"spectator"`（不区分大小写） |
| `getReach()` | 数字 | 玩家当前的触及（交互）距离 |

!!! warning "setGameMode 只改客户端"
    `setGameMode` 修改的是客户端本地状态，服务器并不知道。在服务器上乱用会导致显示与实际不一致，一般只用于单人测试或配合观察类脚本。

### 准心射线

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `rayTraceBlock(distance, fluid)` | `BlockDataHelper` 或 `null` | 玩家准心指向的方块/液体，`fluid=true` 时命中流体 |
| `detailedRayTraceBlock(distance, fluid)` | `HitResultHelper$Block` | 更详细的方块命中结果（命中面、是否落空等） |
| `rayTraceEntity(distance)` | `EntityHelper` 或 `null` | 玩家准心指向的实体，`distance` 为整数 |
| `rayTraceEntity()` | `EntityHelper` 或 `null` | :warning: 已弃用；会受 `setTarget` 影响，请改用带距离的重载或 `interactions().getTargetedEntity()` |

详见下方[射线检测](#射线检测准心检测)小节。

### 告示牌与截图

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `writeSign(l1, l2, l3, l4)` | 布尔 | 向当前打开的告示牌编辑界面写四行文本，某行传 `null` 则该行保持不变；返回是否成功（界面是否打开） |
| `writeSign(index, message)` | 布尔 | 只写某一行，`index` 取 0~3 |
| `takeScreenshot(folder, callback)` | 无 | 截图保存到指定文件夹，完成后回调收到一个 `TextHelper` |
| `takeScreenshot(folder, file, callback)` | 无 | 同上，但指定文件名 |
| `takePanorama(folder, width, height, callback)` | 无 | 全景截图（六个面），`folder` 相对于脚本目录 |

!!! note "回调必须包装"
    截图的 `callback` 是 Java 侧的 `Consumer<TextHelper>`，必须用 `JavaWrapper.methodToJava(...)` 包装（不需要回调时可以传 `null`）：

    ```javascript
    Player.takeScreenshot("screenshots", JavaWrapper.methodToJava((msg) => {
        Chat.log(msg)
    }))
    ```

写告示牌示例（先对着告示牌按右键打开编辑界面，再运行脚本）：

```javascript
const ok = Player.writeSign("第一行", "第二行", "", "")
if (!ok) {
    Chat.log("当前没有打开告示牌编辑界面")
}

// 或者只改第 2 行（下标从 0 开始）
Player.writeSign(1, "只改这一行")
```

### 统计信息

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getStatistics()` | `StatsHelper` | 获取统计信息对象，详见下方[统计信息](#统计信息-statshelper)小节 |

### 移动输入

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `createPlayerInput()` | `PlayerInput` | 创建全 0/false 的空输入 |
| `createPlayerInput(movementForward, movementSideways, yaw)` | `PlayerInput` | 只指定前后、左右、水平朝向 |
| `createPlayerInput(movementForward, yaw, jumping, sprinting)` | `PlayerInput` | 指定前后、朝向、跳跃、疾跑 |
| `createPlayerInput(movementForward, movementSideways, yaw, pitch, jumping, sneaking, sprinting)` | `PlayerInput` | 完整七参数版本 |
| `createPlayerInputsFromCsv(csv)` | `PlayerInput` 列表 | 从 CSV 字符串批量创建（表头大小写敏感） |
| `createPlayerInputsFromJson(json)` | `PlayerInput` | 从 JSON 字符串创建（键名大小写敏感） |
| `getCurrentPlayerInput()` | `PlayerInput` | 把玩家当前这一刻的按键状态抓成一个输入对象 |
| `addInput(input)` | 无 | 把一个输入加入移动队列（每 tick 消耗一个） |
| `addInputs(inputs)` | 无 | 把输入数组依次加入移动队列 |
| `clearInputs()` | 无 | 清空移动队列 |
| `predictInput(input)` / `predictInput(input, draw)` | `Pos3D` | 预测执行该输入一个 tick 后玩家会到哪，`draw=true` 时在世界中可视化 |
| `predictInputs(inputs)` / `predictInputs(inputs, draw)` | `Pos3D` 列表 | 依次预测一串输入每个 tick 后的位置 |
| `setDrawPredictions(val)` | 无 | 开关预测结果的可视化绘制 |
| `moveForward(yaw)` | 无 | 向前走一步（`yaw` 为相对角度），加入移动队列 |
| `moveBackward(yaw)` | 无 | 向后走一步（相对角度） |
| `moveStrafeLeft(yaw)` | 无 | 向左横移一步（相对角度） |
| `moveStrafeRight(yaw)` | 无 | 向右横移一步（相对角度） |
| `isBreakingBlock()` | 布尔 | :warning: 已弃用，请改用 `Player.getInteractionManager().isBreakingBlock()` |

用法详见下方[移动输入教学](#移动输入教学)小节。

!!! note "createPos 这些去哪了？"
    在 2.1.0 的类型定义中，`createPos(x, y, z)`、`createPos(x, y)`、`createBlockPos(x, y, z)`、`createVec(...)`、`createLookingVector(entity)`、`createLookingVector(yaw, pitch)` 属于全局库 **`PositionCommon`**，不在 `Player` 下：

    ```javascript
    const pos = PositionCommon.createPos(0, 64, 0)
    const blockPos = PositionCommon.createBlockPos(0, 64, 0)
    const vec = PositionCommon.createVec(0, 64, 0, 10, 64, 10)
    const looking = PositionCommon.createLookingVector(Player.getPlayer())
    ```

    详细用法见[位置与向量](position.md)。

## 射线检测（准心检测）

射线（ray trace）就是"从玩家眼睛沿准心方向发射一条线，看打到什么"。

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

- `rayTraceBlock(distance, fluid)` 的 `fluid=true` 时会把水、岩浆等流体也算作命中目标。
- `rayTraceEntity(distance)` 的距离参数是整数。

需要知道"命中的是哪个面"或"到底有没有打中"时，用 `detailedRayTraceBlock`：

```javascript
const hit = Player.detailedRayTraceBlock(20, false)
if (!hit.isMissed()) {
    const pos = hit.getBlockPos()
    Chat.log(`命中方块: ${pos.getX()}, ${pos.getY()}, ${pos.getZ()}`)
    Chat.log(`命中面: ${hit.getSide().getName()}`)
    Chat.log(`精确命中点: ${hit.getPos()}`)
} else {
    Chat.log(`${20}格内没有方块`)
}
```

`HitResultHelper$Block` 常用方法：

| 方法 | 说明 |
| --- | --- |
| `getPos()` | 精确命中坐标（`Pos3D`，不是方块坐标） |
| `getBlockPos()` | 命中方块的整数坐标（可能为 `null`） |
| `getSide()` | 命中的面（`DirectionHelper`，可能为 `null`） |
| `isMissed()` | 是否什么都没打中 |
| `isInsideBlock()` | 命中点是否在方块内部 |

!!! tip "配合交互管理器"
    "看着的实体"也可以用 `Player.interactions().getTargetedEntity()` 获取，它与游戏自己的目标选择逻辑一致。详见[交互管理器](interaction.md)。

## 移动输入教学

移动输入（`PlayerInput`）是"替玩家按键"的机制：你创建一串输入对象放进移动队列（MovementQueue），游戏**每个 tick 消耗一个**，就像玩家真的按下了那些键。适合做短暂、可预测的移动，不适合无脑高速循环。

### PlayerInput 参数表

`createPlayerInput` 完整七参数版本：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `movementForward` | 数字 | `1` = 前进(W)，`0` = 不动，`-1` = 后退(S)，允许小数 |
| `movementSideways` | 数字 | `1` = 向左(A)，`0` = 不动，`-1` = 向右(D)，允许小数 |
| `yaw` | 数字 | 水平朝向角（**绝对**角度，不是相对） |
| `pitch` | 数字 | 俯仰角（绝对角度） |
| `jumping` | 布尔 | 是否按住跳跃 |
| `sneaking` | 布尔 | 是否按住潜行 |
| `sprinting` | 布尔 | 是否疾跑 |

!!! warning "容易搞反"
    `movementSideways` 的 `1` 是**向左**，`-1` 才是向右——与很多人的直觉相反，写错了人就往反方向走。

### 基本用法：前进一秒

一个输入只管一个 tick，想走 1 秒就要加 20 个：

```javascript
const player = Player.getPlayer()
if (!player) return

const input = Player.createPlayerInput(1, 0, player.getYaw())
for (let i = 0; i < 20; i++) {
    Player.addInput(input)
}
```

中途想取消，清空队列即可：

```javascript
Player.clearInputs()
```

快捷方法（`yaw` 为**相对**当前朝向的角度，一次只加一步）：

```javascript
Player.moveForward(0)      // 朝当前方向前进一步
Player.moveBackward(0)     // 后退一步
Player.moveStrafeLeft(0)   // 左横移一步
Player.moveStrafeRight(0)  // 右横移一步
```

### 预测：先算后走

`predictInput` 可以在不真的移动的情况下，算出"执行这个输入一个 tick 后玩家会在哪"：

```javascript
const player = Player.getPlayer()
if (!player) return

const input = Player.createPlayerInput(1, 0, player.getYaw())
const predicted = Player.predictInput(input)
Chat.log(`下一 tick 大约在: ${predicted.getX().toFixed(2)}, ${predicted.getY().toFixed(2)}, ${predicted.getZ().toFixed(2)}`)
```

一串输入用 `predictInputs`，还可以让 JsMacros 把预测轨迹画在世界里：

```javascript
const inputs = []
for (let i = 0; i < 20; i++) {
    inputs.push(Player.createPlayerInput(1, 0, Player.getPlayer().getYaw()))
}
const path = Player.predictInputs(inputs, true) // true = 可视化
Chat.log(`20 tick 后大约到达: ${path[path.length - 1]}`)
```

`setDrawPredictions(true/false)` 可以全局开关这个可视化。

### 批量创建：CSV 与 JSON

```javascript
// CSV：第一行是表头（大小写敏感），之后每行一个 tick 的输入
const inputs = Player.createPlayerInputsFromCsv(
    "movementForward, movementSideways, yaw\n" +
    "1, 0, 0\n" +
    "1, 0, 45\n" +
    "1, 0, 90"
)
for (const input of inputs) {
    Player.addInput(input)
}

// JSON：键名同样大小写敏感
const single = Player.createPlayerInputsFromJson('{"movementForward": 1, "yaw": 90, "jumping": true}')
Player.addInput(single)
```

`getCurrentPlayerInput()` 能把你此刻的真实按键抓成一个 `PlayerInput`，配合 `toString(true)`（JSON）/`toString(false)`（CSV）可以做"录制—回放"：

```javascript
const now = Player.getCurrentPlayerInput()
Chat.log(now.toString(true)) // 以 JSON 形式打印当前输入
```

!!! question "跳跃前进"
    用移动输入让玩家：先向前走 10 tick，然后边前进边跳一下，再继续向前走 10 tick。

    需要方法:
    > `Player.createPlayerInput(movementForward, movementSideways, yaw)` 创建普通前进输入。

    > `Player.createPlayerInput(movementForward, yaw, jumping, sprinting)` 创建带跳跃的输入。

    > `Player.addInput(input)` 把输入加入队列。

    ???- tip "答案"
        ```javascript
        const player = Player.getPlayer()
        if (!player) return
        const yaw = player.getYaw()

        for (let i = 0; i < 10; i++) {
            Player.addInput(Player.createPlayerInput(1, 0, yaw))
        }
        Player.addInput(Player.createPlayerInput(1, yaw, true, false)) // 这一 tick 按下跳跃
        for (let i = 0; i < 10; i++) {
            Player.addInput(Player.createPlayerInput(1, 0, yaw))
        }
        ```
        队列每 tick 消耗一个输入，所以"跳一下"就是插入一个 `jumping=true` 的输入。
        实际起跳后玩家会在空中飞行几个 tick，后面的前进输入会在空中继续生效。

## 统计信息 StatsHelper

`Player.getStatistics()` 返回 `StatsHelper`，对应游戏里 Esc → 统计信息 那个界面的数据。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `updateStatistics()` | 自身 | 向服务器请求刷新统计数据（数据是服务器发来的，先刷新再读更准） |
| `getStatList()` | 字符串列表 | 所有可用统计项的 key |
| `getRawStatValue(statKey)` | 数字 | 某项统计的原始数值 |
| `getFormattedStatValue(statKey)` | 字符串 | 某项统计的格式化数值（如把 cm 换算成 km） |
| `getRawStatMap()` | Map | 全部统计项的原始值 |
| `getFormattedStatMap()` | Map | 全部统计项的格式化值 |
| `getStatText(statKey)` | 原版文本对象 | 某项统计的显示名（较少用） |
| `getEntityKilled(id)` | 数字 | 击杀某种实体的次数 |
| `getKilledByEntity(id)` | 数字 | 被某种实体击杀的次数 |
| `getBlockMined(id)` | 数字 | 挖掘某种方块的次数 |
| `getItemBroken(id)` | 数字 | 用坏某种物品的次数 |
| `getItemCrafted(id)` | 数字 | 合成某种物品的次数 |
| `getItemUsed(id)` | 数字 | 使用某种物品的次数 |
| `getItemPickedUp(id)` | 数字 | 捡起某种物品的次数 |
| `getItemDropped(id)` | 数字 | 丢弃某种物品的次数 |
| `getCustomStat(id)` | 数字 | 自定义统计项原始值（如 `"jump"` 跳跃次数） |
| `getCustomFormattedStat(id)` | 字符串 | 自定义统计项格式化值 |

```javascript
const stats = Player.getStatistics()
stats.updateStatistics()   // 请求服务器刷新
Time.sleep(1000)           // 稍等数据回来

Chat.log(`击杀苦力怕: ${stats.getEntityKilled("minecraft:creeper")}`)
Chat.log(`挖掘石头: ${stats.getBlockMined("minecraft:stone")}`)
Chat.log(`吃掉面包: ${stats.getItemUsed("minecraft:bread")}`)
```

## 本地玩家完整参考：ClientPlayerEntityHelper

`Player.getPlayer()` 返回的就是 `ClientPlayerEntityHelper`。它的继承链：

```
ClientPlayerEntityHelper → PlayerEntityHelper → LivingEntityHelper → EntityHelper
```

也就是说，玩家对象同时拥有下面几张表里的**全部**方法（其他玩家实体只有 `PlayerEntityHelper` 及以下的方法，见[实体](entities.md)）。

### 朝向控制

| 方法 | 说明 |
| --- | --- |
| `lookAt(direction)` | 朝某个正方向看，`direction` 可为 `"up"` / `"down"` / `"north"` / `"south"` / `"east"` / `"west"`，只改对应轴 |
| `lookAt(yaw, pitch)` | 设置绝对朝向角 |
| `lookAt(x, y, z)` | 看向指定坐标 |
| `tryLookAt(x, y, z)` | 尝试多种角度让准心命中指定方块，成功转头并返回 `true`，失败保持原朝向返回 `false` |
| `tryLookAt(pos)` | 同上，参数为 `BlockPosHelper` |
| `turnLeft()` | 左转 90 度 |
| `turnRight()` | 右转 90 度 |
| `turnBack()` | 转身 180 度 |

```javascript
const player = Player.getPlayer()
if (!player) return

player.lookAt(0, -60, 0)        // 看向坐标
Time.sleep(200)
player.turnBack()               // 转身
Time.sleep(200)
if (player.tryLookAt(0, -61, 0)) {
    Chat.log("准心已对准方块，可以放心交互")
    Player.interactions().interact()
}
```

!!! tip "转头 + interact 更像人"
    比起直接 `interactEntity`，"先 `lookAt`/`tryLookAt` 转头、再 `interactions().interact()`"产生的发包顺序更接近真人操作，在有反作弊的服务器上更安全。

### 位置与速度（客户端侧）

| 方法 | 说明 |
| --- | --- |
| `setPos(x, y, z)` / `setPos(pos)` | 直接设置玩家坐标 |
| `setPos(x, y, z, await)` / `setPos(pos, await)` | 同上，`await=true` 时等待操作在游戏线程完成 |
| `addPos(x, y, z)` / `addPos(pos)` | 在当前坐标上偏移 |
| `addPos(x, y, z, await)` / `addPos(pos, await)` | 同上，可等待 |
| `setVelocity(x, y, z)` / `setVelocity(velocity)` | 设置速度向量 |
| `addVelocity(x, y, z)` / `addVelocity(velocity)` | 叠加速度向量 |

```javascript
const player = Player.getPlayer()
player.addVelocity(0, 0.5, 0) // 向上弹一下
```

!!! warning "服务器会把你拉回来"
    这些方法只改**客户端**的位置/速度。在服务器上，超出正常移动范围会被服务器拉回（rubber-band），更大概率直接触发反作弊。它们主要用于单人游戏、位移微调或测试。

### 饥饿与饱和

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getFoodLevel()` | 数字 | 饥饿值（0~20） |
| `getSaturation()` | 数字 | 饱和度（就是一些模组里黄色覆盖层显示的隐藏值） |

```javascript
const player = Player.getPlayer()
if (player.getFoodLevel() <= 6) {
    Chat.log("§c饿了，该吃东西了！")
}
```

### 物品冷却

末影珍珠、紫颂果、盾牌等物品使用后有冷却，这几个方法可以读取：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getItemCooldownsRemainingTicks()` | Map | 所有处于冷却中的物品及剩余 tick |
| `getItemCooldownRemainingTicks(item)` | 数字 | 指定物品剩余冷却 tick |
| `getTicksSinceCooldownsStart()` | Map | 所有冷却物品自冷却开始经过的 tick |
| `getTicksSinceCooldownStart(item)` | 数字 | 指定物品自冷却开始经过的 tick |

```javascript
const player = Player.getPlayer()
const cd = player.getItemCooldownRemainingTicks("ender_pearl")
if (cd > 0) {
    Chat.log(`珍珠冷却还剩 ${cd} tick`)
}
```

### 实用工具

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `dropHeldItem(dropStack)` | 自身 | 丢出手中物品，`true` 丢整组（Ctrl+Q），`false` 丢一个（Q） |
| `getAdvancementManager()` | `AdvancementManagerHelper` | 进度/成就管理器 |
| `calculateMiningSpeed(block)` | 数字 | 估算当前手持物品挖掘某方块（`BlockStateHelper`）需要的 tick 数 |
| `calculateMiningSpeed(usedItem, blockState)` | 数字 | 估算用指定物品挖掘某方块需要的 tick 数（用空气可算空手速度），结果可能差几个 tick 但不会偏大 |

### 已迁移到交互管理器的旧方法

下面这些方法在 2.1.0 中仍然可用（本页教程也在用），但类型定义里都标了"已迁移"。写新脚本请用右列的写法，详见[交互管理器](interaction.md)：

| 旧方法（`player.` 上） | 新写法（`Player.interactions().` 上） |
| --- | --- |
| `attack()` / `attack(await)` | `attack()` |
| `attack(entity)` / `attack(entity, await)` | `attack(entity)` |
| `attack(x, y, z, direction)` 及带 `await` 重载 | `attack(x, y, z, direction)` |
| `interact()` / `interact(await)` | `interact()` |
| `interactEntity(entity, offHand)` 及带 `await` 重载 | `interactEntity(entity, offHand)` |
| `interactItem(offHand)` 及带 `await` 重载 | `interactItem(offHand)` |
| `interactBlock(x, y, z, direction, offHand)` 及带 `await` 重载 | `interactBlock(x, y, z, direction, offHand)` |
| `setLongAttack(stop)` | `breakBlock(...)` 系列 |
| `setLongInteract(stop)` | `holdInteract(...)` 系列 |

坐标类重载的 `direction` 除了 `'up'` 等字符串，也接受 0~5 的数字，顺序为 `[down, up, north, south, west, east]`；带 `await` 的重载会等动作完成后才返回。

## PlayerEntityHelper 方法表

`PlayerEntityHelper` 是所有玩家（包括**其他玩家**）共有的方法。用 `World.getPlayers()` 等方式拿到的其他玩家就是这一层类型，见[实体](entities.md)。

| 分组 | 方法 | 说明 |
| --- | --- | --- |
| 基本 | `getPlayerName()` | 玩家真实名字（字符串，不是显示名） |
| | `getAbilities()` | 能力对象，见下方 `PlayerAbilitiesHelper` |
| | `getScore()` | 玩家分数 |
| 装备 | `getMainHand()` / `getOffHand()` | 主手/副手物品（`ItemStackHelper`） |
| | `getHeadArmor()` / `getChestArmor()` / `getLegArmor()` / `getFootArmor()` | 四件护甲 |
| 经验 | `getXP()` | 总经验值 |
| | `getXPLevel()` | 当前等级 |
| | `getXPProgress()` | 当前等级进度（0~1） |
| | `getXPToLevelUp()` | 升到下一级还需要的经验 |
| 状态 | `isSleeping()` | 是否在睡觉 |
| | `isSleepingLongEnough()` | 睡的时间是否已足够跳过夜晚 |
| 战斗 | `getAttackCooldownProgress()` | 攻击冷却进度（0~1，等于 1 时是满蓄力攻击） |
| | `getAttackCooldownProgressPerTick()` | 每 tick 恢复的冷却进度 |
| 钓鱼 | `getFishingBobber()` | 玩家的鱼漂实体，没在钓鱼时为 `null` |

```javascript
const player = Player.getPlayer()
if (!player) return

Chat.log(`等级 ${player.getXPLevel()}，升级还差 ${player.getXPToLevelUp()} 经验`)

// 满蓄力才攻击（PVP常用）
if (player.getAttackCooldownProgress() >= 1) {
    Player.interactions().attack()
}

// 自动钓鱼的核心判断之一
if (player.getFishingBobber()) {
    Chat.log("正在钓鱼中……")
}
```

## PlayerAbilitiesHelper 方法表

通过 `player.getAbilities()` 获取，对应创造飞行等"能力"状态。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getInvulnerable()` | 布尔 | 是否无敌（不会受伤） |
| `getFlying()` | 布尔 | 当前是否正在飞行 |
| `getAllowFlying()` | 布尔 | 是否允许飞行 |
| `getCreativeMode()` | 布尔 | 是否创造模式 |
| `canModifyWorld()` | 布尔 | 是否允许改动世界（放/挖/交互方块；即使为 `true` 也可能被插件另行限制） |
| `setFlying(b)` | 自身 | 设置飞行状态 |
| `setAllowFlying(b)` | 自身 | 设置是否允许飞行 |
| `getFlySpeed()` | 数字 | 飞行速度倍率 |
| `setFlySpeed(flySpeed)` | 自身 | 设置飞行速度倍率 |
| `getWalkSpeed()` | 数字 | 行走速度 |
| `setWalkSpeed(speed)` | 自身 | 设置行走速度 |

```javascript
const abilities = Player.getPlayer().getAbilities()
Chat.log(`正在飞行: ${abilities.getFlying()}，飞行速度: ${abilities.getFlySpeed()}`)
```

!!! warning "setter 大多只在客户端生效"
    `setFlying`、`setFlySpeed` 这类修改在服务器上会被服务器状态覆盖或触发反作弊，主要用于单人游戏。

## 常用信息速查（继承自实体类）

玩家对象还继承了 `LivingEntityHelper` 和 `EntityHelper` 的大量方法，完整列表见[实体](entities.md)。最常用的一批：

| 方法 | 作用 |
| --- | --- |
| `getPos()` / `getBlockPos()` / `getEyePos()` / `getChunkPos()` | 各种坐标 |
| `getX()` / `getY()` / `getZ()` | 单独取坐标分量 |
| `getYaw()` / `getPitch()` | 朝向角 |
| `getVelocity()` / `getSpeed()` | 速度向量 / 每秒移动格数 |
| `getHealth()` / `getMaxHealth()` / `getAbsorptionHealth()` / `getArmor()` | 血量与护甲 |
| `getStatusEffects()` / `hasStatusEffect(id)` | 状态效果 |
| `isSneaking()` / `isSprinting()` / `isOnGround()` / `isFallFlying()` | 潜行 / 疾跑 / 在地面 / 鞘翅飞行中 |
| `isInLava()` / `isOnFire()` / `getAir()` / `getMaxAir()` | 环境危险状态 |
| `getBiome()` / `getChunk()` | 所在生物群系 / 区块 |
| `distanceTo(...)` | 到实体、方块坐标或坐标点的距离 |
| `getNBT()` | NBT 数据 |
| `rayTraceBlock(distance, fluid)` / `rayTraceEntity(distance)` | 从该实体视角发射线 |

```javascript
const player = Player.getPlayer()
if (!player) return

Chat.log(player.getPlayerName())
Chat.log(`${player.getX().toFixed(1)}, ${player.getY().toFixed(1)}, ${player.getZ().toFixed(1)}`)
Chat.log(`血量: ${player.getHealth()} / ${player.getMaxHealth()}`)
Chat.log(`饥饿: ${player.getFoodLevel()}  经验等级: ${player.getXPLevel()}`)
Chat.log(`潜行: ${player.isSneaking()}  疾跑: ${player.isSprinting()}`)
```

## 相关页面

- [交互管理器](interaction.md)——攻击、挖掘、交互的新写法
- [实体](entities.md)——`EntityHelper` / `LivingEntityHelper` 完整方法
- [背包](inventory.md)——`Player.openInventory()` 返回的 `Inventory` 用法
- [位置与向量](position.md)——`PositionCommon`、`Pos3D`、`BlockPosHelper` 详解
