---
icon: lucide/mouse-pointer-click
---

# 交互管理器 InteractionManagerHelper

交互管理器（`InteractionManagerHelper`）是 JsMacros 2.x 中**攻击、右键交互、挖掘方块、准心目标控制**的统一入口。旧的 `player.attack()`、`player.interactBlock()` 等方法在类型文件中已全部标注 `@deprecated`，并注明"moved to `Player.getInteractionManager()`"——新脚本一律推荐用本页的写法。

它的核心思路是：管理器内部维护一个"当前目标"（准心指向的方块/实体，或你用 `setTarget()` 手动覆盖的目标），无参数的 `attack()`、`interact()`、`breakBlock()` 都作用于这个目标；带坐标/实体参数的重载则是"临时指定目标 → 执行 → 恢复"的快捷方式。

## 获取对象

两个入口完全等价，`Player.interactions()` 只是 `Player.getInteractionManager()` 的别名（类型文件 JSDoc 原文："alias for `Player.getInteractionManager()`"），都自 1.9.0 起可用：

```javascript
const im = Player.getInteractionManager()
// 或
const im2 = Player.interactions()
```

!!! warning "可能返回 null"
    没进入世界（例如还在主菜单）时两者都返回 `null`。写在事件回调或服务里的代码务必先判空：

    ```javascript
    const im = Player.interactions()
    if (!im) throw "当前不在游戏世界中，交互管理器不可用"
    Chat.log("当前游戏模式：" + im.getGameMode())
    ```

!!! tip "支持链式调用"
    绝大多数方法返回管理器自身，可以连写：

    ```javascript
    Player.interactions().setTarget(0, 64, 0, "up").interact()
    ```

## 从旧方法迁移

以下 `player` 指 `Player.getPlayer()` 返回的玩家对象。旧方法目前仍能用，但已弃用，随时可能移除：

| 旧写法（已弃用） | 新写法 |
| --- | --- |
| `player.attack()` / `player.attack(await)` | `im.attack()` / `im.attack(await)` |
| `player.attack(entity[, await])` | `im.attack(entity[, await])` |
| `player.attack(x, y, z, direction[, await])` | `im.attack(x, y, z, direction[, await])` |
| `player.interact()` / `player.interact(await)` | `im.interact()` / `im.interact(await)` |
| `player.interactEntity(entity, offHand[, await])` | `im.interactEntity(entity, offHand[, await])` |
| `player.interactItem(offHand[, await])` | `im.interactItem(offHand[, await])` |
| `player.interactBlock(x, y, z, direction, offHand[, await])` | `im.interactBlock(x, y, z, direction, offHand[, await])` |
| `player.setLongAttack(stop)`（长按左键） | `im.breakBlock()` / `im.breakBlockAsync(callback)` |
| `player.setLongInteract(stop)`（长按右键） | `im.holdInteract(...)` |
| `Player.isBreakingBlock()` | `im.isBreakingBlock()` |
| `Player.rayTraceEntity()`（无参版） | `im.getTargetedEntity()` 或 `Player.rayTraceEntity(distance)` |

!!! note
    `interactEntity` / `interactItem` / `interactBlock` 这几个名字是 1.6.0 从 `interact` 拆分改名而来，1.9.0 起整体搬进了交互管理器。

## 目标：查询与覆盖

### 设置目标 setTarget

`setTarget()` 会**覆盖准心目标**：之后的 `attack()`、`interact()`、`breakBlock()`（无参版本）都作用于这个覆盖目标，直到你调用 `clearTargetOverride()` 清除。

| 方法 | 说明 |
| --- | --- |
| `setTarget(x, y, z)` | 目标设为指定坐标的方块 |
| `setTarget(x, y, z, direction)` | 同上并指定命中面 |
| `setTarget(pos)` | 用 `BlockPosHelper` 指定方块 |
| `setTarget(pos, direction)` | 同上并指定命中面 |
| `setTarget(entity)` | 目标设为实体（`EntityHelper`） |
| `setTargetMissed()` | 目标设为"未命中"（什么都不指着） |

`direction` 参数两种写法都可以：

| 写法 | 取值 |
| --- | --- |
| 字符串 | `"up"`、`"down"`、`"north"`、`"south"`、`"east"`、`"west"` |
| 数字 0–5 | 依次为 `DOWN, UP, NORTH, SOUTH, WEST, EAST` |

### 查询目标

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getTarget()` | `HitResultHelper` 或 `null` | 当前命中结果（1.9.1+） |
| `getTargetedBlock()` | `BlockPosHelper` 或 `null` | 当前目标方块坐标，没指着方块时为 `null` |
| `getTargetedEntity()` | `EntityHelper` 或 `null` | 当前目标实体，没指着实体时为 `null` |
| `hasTargetOverride()` | `boolean` | 是否处于 `setTarget()` / `setTargetMissed()` 的覆盖状态 |
| `clearTargetOverride()` | 自身 | 清除覆盖，恢复真实准心目标 |

`getTarget()` 返回的 `HitResultHelper` 常用方法：

| 方法 | 说明 |
| --- | --- |
| `getPos()` | 命中点精确坐标（`Pos3D`） |
| `asBlock()` | 转为方块命中结果，不是方块则为 `null`；其上有 `getBlockPos()`、`getSide()`、`isMissed()`、`isInsideBlock()` |
| `asEntity()` | 转为实体命中结果，不是实体则为 `null`；其上有 `getEntity()` |

示例——覆盖目标为脚下方块并读取信息：

```javascript
const im = Player.interactions()
if (!im) throw "交互管理器不可用"

const pos = Player.getPlayer().getBlockPos().down()
im.setTarget(pos, "up")

const hit = im.getTarget()
const block = hit ? hit.asBlock() : null
if (block) {
    Chat.log("目标方块：" + block.getBlockPos() + "，命中面：" + block.getSide())
}
Chat.log("是否有目标覆盖：" + im.hasTargetOverride())

im.clearTargetOverride() // 用完记得清除
```

### 目标有效性检查

覆盖目标默认会做一些"合理性"检查，可以按需开关。每个方法都有两个参数：`enabled` 是否启用检查，`autoClear` 检查不通过时是**自动清除覆盖**（`true`）还是把目标**置为未命中**（`false`）：

| 方法 | 检查内容 | 默认值 |
| --- | --- | --- |
| `setTargetRangeCheck(enabled, autoClear)` | 目标是否超出触及距离 | `enabled=true`，`autoClear=true` |
| `setTargetAirCheck(enabled, autoClear)` | 目标方块是否为空气 | `enabled=false`，`autoClear=false` |
| `setTargetShapeCheck(enabled, autoClear)` | 目标方块是否没有形状（此检查忽略空气，空气交给上面那项） | `enabled=true`，`autoClear=false` |
| `resetTargetChecks()` | 把以上全部恢复默认 | — |

```javascript
const im = Player.interactions()
if (!im) throw "交互管理器不可用"

im.setTargetRangeCheck(true, true)   // 超距离时自动清除覆盖
im.setTargetAirCheck(true, false)    // 变成空气时置为"未命中"
im.setTargetShapeCheck(true, true)   // 没有形状时自动清除

// ……做完事之后恢复默认
im.resetTargetChecks()
```

!!! tip
    玩家当前触及距离可以用 `Player.getReach()` 查询，详见[玩家](player.md)页。

## 攻击

对应"按左键一下"。所有重载都可以在最后追加一个 `await` 布尔参数表示等动作完成后再返回（见[await 参数与线程](#await)）：

| 方法 | 说明 |
| --- | --- |
| `attack()` / `attack(await)` | 攻击当前目标（准心指向或 `setTarget` 覆盖的目标） |
| `attack(entity)` / `attack(entity, await)` | 攻击指定实体 |
| `attack(x, y, z, direction)` / `attack(x, y, z, direction, await)` | 对指定坐标方块的某个面挥击一下 |

对准心目标攻击：

```javascript
// 等价于按一下左键，攻击准心指向的实体或敲一下方块
Player.interactions().attack()
```

攻击附近的实体：

```javascript
const im = Player.interactions()
if (!im) throw "交互管理器不可用"

const zombies = World.getEntities(4.5, "minecraft:zombie")
if (zombies && !zombies.isEmpty()) {
    im.attack(zombies.get(0), true) // await = true：等这一下攻击完成再往下走
    Chat.log("已攻击最近的僵尸")
} else {
    Chat.log("4.5 格内没有僵尸")
}
```

攻击指定坐标方块的某个面：

```javascript
// 对 (0, 64, 0) 方块的顶面挥一下手（只打一下，不会连续挖掘）
Player.interactions().attack(0, 64, 0, "up")
```

!!! note "attack 不等于挖掘"
    `attack()` 对方块只是"点一下左键"。要把方块**挖到碎**，请用下文的 [`breakBlock()`](#breakblock) —— 它会自动按住直到方块被破坏。

## 交互（右键）

对应"按右键一下"。`offHand` 参数：`false` = 主手，`true` = 副手。同样支持末尾追加 `await`：

| 方法 | 说明 |
| --- | --- |
| `interact()` / `interact(await)` | 对当前目标按一下右键 |
| `interactEntity(entity, offHand)` / `interactEntity(entity, offHand, await)` | 右键指定实体 |
| `interactItem(offHand)` / `interactItem(offHand, await)` | 使用手中物品（吃食物、扔雪球、喝药水等） |
| `interactBlock(x, y, z, direction, offHand)` / `interactBlock(x, y, z, direction, offHand, await)` | 右键指定坐标方块的某个面 |

```javascript
const im = Player.interactions()
if (!im) throw "交互管理器不可用"

im.interact()                            // 对准心目标右键（开箱子、按按钮……）
im.interactItem(false)                   // 使用主手物品
im.interactBlock(0, 64, 0, "up", false)  // 右键 (0, 64, 0) 方块的顶面
```

!!! warning "反作弊：interactEntity 有发包顺序问题"
    `interactEntity()` 产生的数据包顺序与原版客户端不同（缺少 InteractAt / 主手交互包），会被 GrimAC 的 PacketOrderC / PacketOrderD 检测到——**即使目标就在你面前也一样**。详细分析见[玩家页的 interactEntity 小节](player.md#interactentityentity-offhand)。
    更安全的替代：先用 `lookAt()` / `tryLookAt()` 把准心转向目标，再调用 `interactions().interact()`。

### 按住右键 holdInteract

模拟"长按右键"：拉弓、格挡、吃东西、钓鱼等都需要按住而不是点一下。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `holdInteract(holding)` | 自身 | `true` 开始按住右键，`false` 松开 |
| `holdInteract(holding, awaitFirstClick)` | 自身 | 同上；`awaitFirstClick` 控制是否等第一次使用动作发出后再返回 |
| `holdInteract(ticks)` | `number` | 按住指定 tick 数（阻塞直到完成）；若中途被打断，返回剩余 tick 数，正常完成返回 0 |
| `holdInteract(ticks, stopOnPause)` | `number` | 同上；`stopOnPause=false` 时游戏暂停不会中止本次按住——计时器不减少，恢复后继续按满指定 tick |
| `hasInteractOverride()` | `boolean` | `holdInteract` 的长按是否正在生效 |

示例——手持钓鱼竿，按住右键 2 秒（40 tick）后松开（抛竿）：

```javascript
const im = Player.interactions()
if (!im) throw "交互管理器不可用"

const remaining = im.holdInteract(40) // 阻塞 2 秒
if (remaining > 0) {
    Chat.log("按住被提前打断，还剩 " + remaining + " tick")
} else {
    Chat.log("完整按住了 2 秒")
}
```

等价的手动控制写法（适合中途要做别的判断时）：

```javascript
const im = Player.interactions()
if (!im) throw "交互管理器不可用"

im.holdInteract(true)   // 按下右键不放
Client.waitTick(40)     // 保持 2 秒
im.holdInteract(false)  // 松开
```

## 挖掘方块 {#breakblock}

`breakBlock()` 是"按住左键直到方块碎掉"的完整流程，同步版本会**阻塞脚本直到挖完**：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `breakBlock()` | `BreakBlockResult` 或 `null` | 挖当前目标方块（可先 `setTarget()` 指定），管理器不可用时返回 `null` |
| `breakBlock(x, y, z)` | `BreakBlockResult` 或 `null` | 挖指定坐标的方块 |
| `breakBlock(pos)` | `BreakBlockResult` 或 `null` | 用 `BlockPosHelper` 指定 |
| `breakBlockAsync(callback)` | 自身 | 开始挖掘后立即返回，挖完时调用回调（回调可传 `null`） |
| `isBreakingBlock()` | `boolean` | 当前是否正在挖方块 |
| `hasBreakBlockOverride()` | `boolean` | 是否还有未完成的 `breakBlock()` 任务 |
| `cancelBreakBlock()` | 自身 | 取消由 `breakBlock()` / `breakBlockAsync()` 发起的挖掘 |

坐标版 `breakBlock(x, y, z)` 在类型文件中给出了等价逻辑——先 `setTarget(x, y, z)`，确认目标确实是这个方块后再挖，最后自动 `clearTargetOverride()`，所以不用担心它残留目标覆盖。

### 挖掘结果 BreakBlockResult

结果对象有两个只读**字段**（注意是字段不是方法，直接 `result.reason` 访问）：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `reason` | 字符串或 `null` | 结束原因，见下表 |
| `pos` | `BlockPosHelper` 或 `null` | 相关方块位置 |

| reason | 含义 |
| --- | --- |
| `SUCCESS` | 客户端侧已挖掉（不保证服务器接受：领地保护、服务器卡顿都可能回滚） |
| `CANCELLED` | 调用了 `cancelBreakBlock()` |
| `INTERRUPTED` | 被攻击键（默认左键）打断 |
| `NOT_BREAKING` | 挖掘因某种原因无效 |
| `RESET` | 交互代理被重置或不在世界中 |
| `NO_OVERRIDE` | 挖掘覆盖已失效但仍有残留回调 |
| `IS_AIR` | 目标方块是空气 |
| `NO_TARGET` | 没有目标方块 |
| `TARGET_LOST` | 目标方块丢失 |
| `TARGET_CHANGE` | 目标方块发生变化 |
| `UNAVAILABLE` | 交互管理器不可用 |
| `null` | 未知（代理方法被 API 之外的代码调用） |

同步挖掘脚下方块：

```javascript
const im = Player.interactions()
if (!im) throw "交互管理器不可用"

const pos = Player.getPlayer().getBlockPos().down()
const result = im.breakBlock(pos) // 阻塞直到挖完或失败
if (result) {
    Chat.log("挖掘结束，原因：" + result.reason + (result.pos ? "，位置：" + result.pos : ""))
}
```

异步挖掘 + 中途取消：

```javascript
const im = Player.interactions()
if (!im) throw "交互管理器不可用"

im.setTarget(Player.getPlayer().getBlockPos().down())
im.breakBlockAsync(JavaWrapper.methodToJavaAsync((result) => {
    Chat.log("异步挖掘结束：" + result.reason)
}))

// 挖掘进行中可以做别的事，也可以随时取消：
Client.waitTick(10)
if (im.isBreakingBlock()) {
    im.cancelBreakBlock()
    Chat.log("已取消挖掘")
}
im.clearTargetOverride()
```

!!! warning "breakBlockAsync 的回调要用 methodToJavaAsync"
    类型文件明确注明：这个回调**大多在主线程上被调用**，必须用 `JavaWrapper.methodToJavaAsync(...)` 而不是 `JavaWrapper.methodToJava(...)` 包装，否则可能报错。

!!! tip "挖掘进度"
    交互管理器本身只提供 `isBreakingBlock()` / `hasBreakBlockOverride()` 两个状态查询，没有百分比进度方法。预估挖掘耗时可以用玩家对象的 `calculateMiningSpeed(blockState)`（返回大约需要的 tick 数），见[玩家](player.md)页。

## await 参数与线程 {#await}

`attack` / `interact` / `interactEntity` / `interactItem` / `interactBlock` 的每个重载都有带 `await: boolean` 的版本：

- **不传或 `false`**：动作提交后立即返回，脚本继续往下跑；
- **`true`**：等这次动作真正执行完成后才返回。

另外这几个方法**本身就是阻塞式**的，类型文件标注了 `@throws InterruptedException`（脚本被外部停止时会中断等待）：

- `breakBlock()` 及其坐标/`BlockPosHelper` 重载——阻塞到方块挖完；
- `holdInteract(ticks)` / `holdInteract(ticks, stopOnPause)`——阻塞指定 tick 数。

!!! warning "不要在主线程上阻塞"
    `await = true`、`breakBlock()`、`holdInteract(ticks)`、`Client.waitTick()` 都会让当前脚本线程等待。如果这段代码恰好运行在主线程（例如用 `methodToJava` 包装、被游戏主线程同步调用的回调里），就会造成循环等待卡死游戏。事件回调里要用这些方法时，用 `JavaWrapper.methodToJavaAsync(...)` 包装让它跑在独立线程上。事件与线程模型详见[事件系统](events.md)。

## 游戏模式与底层管理

| 成员 | 说明 |
| --- | --- |
| `getGameMode()` | 返回当前游戏模式 |
| `setGameMode(gameMode)` | 设置游戏模式，可选 `survival`、`creative`、`adventure`、`spectator`（大小写不敏感），返回自身 |
| `autoUpdateBase` | 布尔**字段**，默认 `true`，见下方说明 |
| `checkBase(update)` | 检查底层管理器是否为最新；`update=true` 时顺便更新，`false` 且不是最新则抛错。返回底层管理器是否可用 |

```javascript
const im = Player.interactions()
if (!im) throw "交互管理器不可用"
Chat.log("当前游戏模式：" + im.getGameMode())
```

!!! note "setGameMode 只影响客户端"
    多人服务器上，服务端不会因为客户端自称换了模式就给你对应权限。这个方法主要用于单人或测试。`Player.setGameMode()` 是同样的道理。

??? info "autoUpdateBase 是干什么的？（进阶）"
    交互管理器 helper 底层包着 MC 的 `MultiPlayerGameMode` 对象，切换世界/服务器后底层对象会换新。`autoUpdateBase` 控制 helper 发现底层对象过期时的行为（默认 `true`）：

    - `false`：直接抛错；
    - `true` 且成功更新：方法照常工作；
    - 方法不需要管理器或网络交互：用旧管理器照常执行；
    - 其他情况：方法什么也不做。

    日常脚本保持默认值即可，长期驻留的服务脚本如果想显式感知世界切换，可配合 `checkBase(true)` 使用。

## 反作弊注意事项

!!! warning "反作弊提醒"
    目标覆盖和直接交互可能产生原版客户端不容易做到的包序列。多人服务器里优先使用准心对准后的普通交互（先 `lookAt` 转头再 `interact()`/`attack()`），避免跨距离、隔墙或过高频操作。

    - `interactEntity()` 存在确定的发包顺序缺陷（GrimAC PacketOrderC / PacketOrderD），细节见[玩家页](player.md#interactentityentity-offhand)；
    - `setTarget()` + `interact()`/`breakBlock()` 指向准心之外的目标时，等于"没看着也能操作"，是各类反作弊的重点检测项；
    - `holdInteract` / `breakBlock` 本身模拟的是正常按键流程，风险主要来自你选的目标和触发频率。

## 相关页面

- [玩家](player.md)——玩家对象、`lookAt` 转头、旧交互方法与反作弊详解
- [方块](blocks.md)——`BlockPosHelper` 与方块数据
- [实体](entities.md)——`EntityHelper` 与实体筛选
- [事件系统](events.md)——事件回调与线程模型
