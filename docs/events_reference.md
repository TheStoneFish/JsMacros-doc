---
icon: lucide/list
---

# 全部事件参考

本页收录 JsMacros 2.1.0（MC 1.21.1）的**全部 59 个事件**，逐一对照 `Events` 命名空间整理，按主题分组。事件机制（on/off、joined、waitForEvent、自定义事件）见 [事件系统](events.md)。

!!! note "阅读须知"
    - **可取消**：事件实现了 `Cancellable` 接口，在 **joined** 监听中调用 `event.cancel()` 可拦截。
    - **适合 joined**：可取消、或有可写字段（joined 时修改才生效）的事件标"是"；其余事件用 joined 只会拖慢游戏，标"否"。
    - 字段类型使用 Helper 简名，如 `ItemStackHelper`；`JavaList<T>` 用 `size()`/`get(i)` 访问（见 [常用类型](types.md)）。
    - `PacketName`、`SoundId`、`ParticleId`、`ScreenName`、`DamageSource`、`Dimension`、`EntityUnloadReason` 等类型在 d.ts 中未展开为字面量，实际是字符串标识，直接当字符串比较即可。
    - 没有列出字段/方法的事件，表示 d.ts 中确实没有（只能用 `getEventName()`）。

**分组导航**

| 分组 | 事件 |
| --- | --- |
| [生命周期与连接](#lifecycle) | LaunchGame、ProfileLoad、JoinServer、Disconnect、QuitGame、Tick、DimensionChange、ResourcePackLoaded |
| [聊天与标题](#chat) | SendMessage、RecvMessage、Title |
| [玩家状态](#player-status) | HealthChange、Damage、Heal、Death、HungerChange、AirChange、EXPChange、ArmorChange、StatusEffectUpdate、FallFlying、Riding |
| [输入与交互](#input) | Key、MouseScroll、InteractBlock、InteractEntity、AttackBlock、AttackEntity |
| [容器与物品](#container) | OpenContainer、ContainerUpdate、ClickSlot、SlotUpdate、DropSlot、HeldItemChange、ItemPickup、ItemDamage |
| [世界](#world) | ChunkLoad、ChunkUnload、BlockUpdate、Sound、ServerSound、Particle |
| [实体](#entity) | EntityLoad、EntityUnload、EntityDamaged、EntityHealed、NameChange、PlayerJoin、PlayerLeave |
| [界面](#screen) | OpenScreen、Bossbar、SignEdit |
| [网络](#network) | RecvPacket、SendPacket |
| [脚本与其他](#script) | Custom、Service、CommandContext、WrappedScript、CodeRender |

## 生命周期与连接 {#lifecycle}

游戏启动、进出服务器、每 tick 心跳。典型示例——进服初始化、断线提示：

```javascript
JsMacros.on("JoinServer", JavaWrapper.methodToJava((e) => {
    Chat.log(`已连接到 ${e.address}，玩家 ${e.player.getName().getString()}`)
}))

JsMacros.on("Disconnect", JavaWrapper.methodToJava((e) => {
    Chat.log("已断开连接")
}))
```

### LaunchGame

游戏客户端启动时触发一次（配置文件加载后）。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `playerName` | `string` | 当前登录的玩家名 |

### ProfileLoad

JsMacros 配置档案（Profile）加载/切换时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `profileName` | `string` | 被加载的档案名 |

### JoinServer

进入服务器（或单人世界）时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `player` | [ClientPlayerEntityHelper](entities.md) | 本地玩家对象 |
| `address` | `string` | 服务器地址 |

### Disconnect

与服务器断开连接时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `message` | [TextHelper](chat.md) | 断开原因文本 |

### QuitGame

退出游戏客户端时触发。**可取消**：否 · **适合 joined**：否

无字段。适合做最后的存档/上报工作（注意时间很短）。

### Tick

每个游戏刻触发一次（每秒 20 次）。**可取消**：否 · **适合 joined**：否

无字段。高频事件，回调里别做重活；配合 `JsMacros.eventFilters().modulus(n)` 可降频。

### DimensionChange

玩家切换维度时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `dimension` | `Dimension`（字符串） | 目标维度 ID，如 `minecraft:the_nether` |

### ResourcePackLoaded

资源包（重新）加载完成时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `isGameStart` | `boolean` | 是否为游戏启动时的首次加载 |
| `loadedPacks` | `JavaList<string>` | 已加载的资源包名列表 |

## 聊天与标题 {#chat}

消息收发与标题显示，全部可取消、可改写，是 joined 监听的主战场。典型示例——聊天过滤：

```javascript
JsMacros.on("RecvMessage", true, JavaWrapper.methodToJava((e, ctx) => {
    if (e.text && e.text.getString().includes("广告")) {
        e.cancel()          // 拦截这条消息
    }
    ctx.releaseLock()       // 放行游戏线程
}))
```

### SendMessage

玩家发送聊天消息/命令**之前**触发。**可取消**：是 · **适合 joined**：是（可取消、可改写）

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `message` | `string`（**可写**，可为 `null`） | 即将发送的内容，joined 中修改可改写发出的消息 |

### RecvMessage

客户端收到聊天消息时触发。**可取消**：是 · **适合 joined**：是（可取消、可改写）

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `text` | [TextHelper](chat.md)（**可写**，可为 `null`） | 消息文本，joined 中替换可改写显示内容 |
| `signature` | `JavaArray<number>` 或 `null` | 消息签名（1.19+ 聊天签名机制） |
| `messageType` | `string` 或 `null` | 消息类型 |

### Title

显示标题/副标题/ActionBar 时触发。**可取消**：是 · **适合 joined**：是（可取消、可改写）

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `type` | `'TITLE'`/`'SUBTITLE'`/`'ACTIONBAR'` | 显示位置 |
| `message` | [TextHelper](chat.md)（**可写**，可为 `null`） | 显示内容 |

## 玩家状态 {#player-status}

血量、饥饿、经验、药水效果等本地玩家的状态变化，全部只读，适合做报警和统计。典型示例——低血量警报：

```javascript
JsMacros.on("HealthChange", JavaWrapper.methodToJava((e) => {
    if (e.health <= 6 && e.change < 0) {
        Chat.log("§c血量告急，快撤！")
    }
}))
```

### HealthChange

玩家血量变化时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `health` | `number` | 变化后的血量 |
| `change` | `number` | 变化量（受伤为负，恢复为正） |

### Damage

玩家受到伤害时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `attacker` | [EntityHelper](entities.md) | 攻击者（*已弃用：多人服务器上可能取不到*） |
| `source` | `DamageSource`（字符串） | 伤害来源（*已弃用：多人服务器上可能取不到*） |
| `health` | `number` | 受伤后的血量 |
| `change` | `number` | 本次变化量 |

### Heal

玩家回血时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `source` | `HealSource`（字符串） | 恢复来源 |
| `health` | `number` | 恢复后的血量 |
| `change` | `number` | 恢复量 |

### Death

玩家死亡时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `deathPos` | [BlockPosHelper](position.md) | 死亡位置 |
| `inventory` | `JavaList<`[ItemStackHelper](inventory.md)`>` | 死亡时携带的物品列表 |

| 方法 | 说明 |
| --- | --- |
| `respawn()` | 重生。应延迟调用（等 1 tick 即可），不要在事件回调里立即执行 |

```javascript
JsMacros.on("Death", JavaWrapper.methodToJava((e) => {
    Chat.log(`死亡位置: ${e.deathPos.toString()}`)
    Client.waitTick(1)
    e.respawn()
}))
```

### HungerChange

饥饿值变化时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `foodLevel` | `number` | 当前饥饿值（0–20） |

### AirChange

氧气值变化（潜水/出水）时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `air` | `number` | 当前氧气值（满值 300） |

### EXPChange

经验变化时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `progress` | `number` | 当前等级进度（0.0–1.0） |
| `total` | `number` | 总经验值 |
| `level` | `number` | 当前等级 |
| `prevProgress` | `number` | 变化前的进度 |
| `prevTotal` | `number` | 变化前的总经验 |
| `prevLevel` | `number` | 变化前的等级 |

### ArmorChange

盔甲槽内容变化时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `slot` | `'HEAD'`/`'CHEST'`/`'LEGS'`/`'FEET'` | 变化的盔甲槽位 |
| `item` | [ItemStackHelper](inventory.md) | 新装备 |
| `oldItem` | [ItemStackHelper](inventory.md) | 旧装备 |

### StatusEffectUpdate

玩家药水效果增删/更新时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `oldEffect` | `StatusEffectHelper` | 变化前的效果 |
| `newEffect` | `StatusEffectHelper` | 变化后的效果 |
| `added` | `boolean` | 是否为新增效果 |
| `removed` | `boolean` | 是否为移除效果 |

### FallFlying

鞘翅滑翔状态切换时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `state` | `boolean` | `true` 开始滑翔，`false` 结束滑翔 |

### Riding

骑乘状态切换（上/下坐骑、载具）时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `state` | `boolean` | `true` 上骑乘，`false` 下骑乘 |
| `entity` | [EntityHelper](entities.md) | 坐骑/载具实体 |

## 输入与交互 {#input}

按键、滚轮与左右键交互。典型示例——自定义快捷键：

```javascript
JsMacros.on("Key", JavaWrapper.methodToJava((e) => {
    if (e.key === "key.keyboard.g" && e.action === 1) {
        Chat.log("按下了 G 键")
    }
}))
```

### Key

键盘/鼠标按键按下或松开时触发。**可取消**：是 · **适合 joined**：是（取消可吞掉按键）

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `action` | `number` | `1` 按下，`0` 松开 |
| `key` | `Key`（字符串） | 按键名，如 `key.keyboard.g`、`key.mouse.left`（见 [按键绑定](keybind.md)） |
| `mods` | `KeyMods` | 修饰键组合，如 `key.keyboard.left.shift`，多个用 `+` 连接（shift+ctrl+alt 顺序） |

### MouseScroll

鼠标滚轮滚动时触发。**可取消**：是 · **适合 joined**：是

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `deltaX` | `number` | 横向滚动量 |
| `deltaY` | `number` | 纵向滚动量（向上为正） |

### InteractBlock

右键与方块交互时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `offhand` | `boolean` | 是否副手交互 |
| `result` | `boolean` | 交互结果是否成功 |
| `block` | [BlockDataHelper](blocks.md) | 目标方块 |
| `side` | `number`（0–5） | 交互的方块面 |

### InteractEntity

右键与实体交互时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `offhand` | `boolean` | 是否副手交互 |
| `result` | `boolean` | 交互结果是否成功 |
| `entity` | [EntityHelper](entities.md) | 目标实体 |

### AttackBlock

攻击（左键）方块时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `block` | [BlockDataHelper](blocks.md) | 被攻击的方块 |
| `side` | `number`（0–5） | 被攻击的方块面 |

### AttackEntity

攻击（左键）实体时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `entity` | [EntityHelper](entities.md) | 被攻击的实体 |

## 容器与物品 {#container}

打开容器、点击槽位、丢物品、切换手持。典型示例——防误丢：

```javascript
JsMacros.on("DropSlot", true, JavaWrapper.methodToJava((e, ctx) => {
    const item = e.getInventory().getSlot(e.slot)
    if (item.getItemId() === "minecraft:diamond_sword") {
        e.cancel() // 拦下这次丢弃
        Chat.log("§e已阻止丢出钻石剑")
    }
    ctx.releaseLock()
}))
```

### OpenContainer

打开容器界面时触发。**可取消**：是 · **适合 joined**：是

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `inventory` | [Inventory](inventory.md) | 容器对应的 Inventory 对象 |
| `screen` | [IScreen](screen.md) | 容器界面 |

### ContainerUpdate

容器内容同步（服务器刷新容器数据）时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `inventory` | [Inventory](inventory.md) | 容器对应的 Inventory 对象 |
| `screen` | [IScreen](screen.md) | 容器界面 |

### ClickSlot

点击槽位（发送点击操作前）时触发。**可取消**：是 · **适合 joined**：是

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `mode` | `number` | 点击模式 0–6（对应协议 Click Window，见 [wiki.vg](https://wiki.vg/Protocol#Click_Window)） |
| `button` | `number` | 按钮编号（鼠标键或数字键等，0–10、40） |
| `slot` | `number` | 被点击的槽位号 |

| 方法 | 说明 |
| --- | --- |
| `getInventory()` | 返回事件关联的 [Inventory](inventory.md) |

### SlotUpdate

槽位内容更新时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `type` | `'HELD'`/`'INVENTORY'`/`'SCREEN'` | 更新区域：光标持有物 / 玩家背包 / 打开的界面 |
| `slot` | `number` | 槽位号 |
| `oldStack` | [ItemStackHelper](inventory.md) | 更新前的物品 |
| `newStack` | [ItemStackHelper](inventory.md) | 更新后的物品 |

| 方法 | 说明 |
| --- | --- |
| `getInventory()` | 返回事件关联的 [Inventory](inventory.md) |

### DropSlot

从槽位丢出物品时触发。**可取消**：是 · **适合 joined**：是

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `slot` | `number` | 槽位号 |
| `all` | `boolean` | `true` 整组丢出（Ctrl+Q），`false` 单个丢出 |

| 方法 | 说明 |
| --- | --- |
| `getInventory()` | 返回事件关联的 [Inventory](inventory.md) |

### HeldItemChange

手持物品切换时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `offHand` | `boolean` | 是否为副手变化 |
| `item` | [ItemStackHelper](inventory.md) | 新的手持物品 |
| `oldItem` | [ItemStackHelper](inventory.md) | 之前的手持物品 |

### ItemPickup

捡起物品时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `item` | [ItemStackHelper](inventory.md) | 捡起的物品 |

### ItemDamage

物品损耗耐久时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `item` | [ItemStackHelper](inventory.md) | 受损的物品 |
| `damage` | `number` | 物品当前损伤值 |

## 世界 {#world}

区块、方块更新与声音、粒子。典型示例——矿物更新提醒：

```javascript
JsMacros.on("BlockUpdate", JavaWrapper.methodToJava((e) => {
    if (e.block.getId() === "minecraft:diamond_ore") {
        Chat.log(`钻石矿更新: ${e.block.getBlockPos().toString()}`)
    }
}))
```

### ChunkLoad

区块加载时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `x` | `number` | 区块 X 坐标（区块坐标，不是方块坐标） |
| `z` | `number` | 区块 Z 坐标 |
| `isFull` | `boolean` | 是否完整加载（含全部方块数据） |

### ChunkUnload

区块卸载时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `x` | `number` | 区块 X 坐标 |
| `z` | `number` | 区块 Z 坐标 |

### BlockUpdate

方块状态或方块实体更新时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `block` | [BlockDataHelper](blocks.md) | 更新后的方块数据 |
| `updateType` | `'STATE'`/`'ENTITY'` | 更新类型：方块状态 / 方块实体 |

### Sound

客户端播放声音时触发。**可取消**：是 · **适合 joined**：是

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `sound` | `SoundId`（字符串） | 声音 ID，如 `minecraft:entity.experience_orb.pickup` |
| `volume` | `number` | 音量 |
| `pitch` | `number` | 音调 |
| `position` | [Pos3D](position.md) | 声源位置 |

### ServerSound

服务器下发声音包时触发。**可取消**：是 · **适合 joined**：是（字段均可写）

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `sound` | `SoundId`（**可写**） | 声音 ID |
| `source` | `SoundCategory`（**可写**） | 声音类别（主音量/音乐/敌对生物等） |
| `position` | [Pos3D](position.md)（**可写**） | 声源位置。`entity` 为 `null` 时坐标按协议精度截断到 1/8 格；否则为实体位置 |
| `entity` | [EntityHelper](entities.md) 或 `null`（**可写**） | 关联实体 |
| `volume` | `number`（**可写**） | 音量 |
| `pitch` | `number`（**可写**） | 音调 |
| `seed` | `number`（**可写**） | 随机种子 |

| 方法 | 说明 |
| --- | --- |
| `getSeed()` | 以字符串返回种子（避免 long 精度问题） |
| `setSeed(seed)` | 以字符串设置种子 |

### Particle

生成粒子（服务器粒子包）时触发。**可取消**：是 · **适合 joined**：是

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `raw` | 原始包对象 | `ClientboundLevelParticlesPacket` |
| `type` | `ParticleId`（字符串） | 粒子 ID，如 `minecraft:dust` |
| `x` / `y` / `z` | `number` | 粒子位置 |
| `offsetX` / `offsetY` / `offsetZ` | `number` | 扩散偏移 |
| `speed` | `number` | 速度 |
| `count` | `number` | 数量 |
| `longDistance` | `boolean` | 是否远距离渲染 |

| 方法 | 说明 |
| --- | --- |
| `getPos()` | 位置 [Pos3D](position.md) |
| `is(id)` | 判断粒子 ID（可省略 `minecraft:` 前缀） |
| `getColor()` | `[r, g, b]`，仅 `dust` 粒子有效，无效时为 `-1` |
| `getHexColor()` | 十六进制颜色，仅 `dust`，无效时 `-1` |
| `getEntityColor()` / `getEntityHexColor()` | `entity_effect` 粒子的颜色 |
| `getTransitionColor()` / `getTransitionHexColor()` | `dust_color_transition` 的起止颜色 |
| `getScale()` | 尘埃类粒子的缩放，无效时 `-1` |
| `getRoll()` | `sculk_charge` 的滚转角，无效时 `-1` |
| `getDelay()` | `shriek` 的延迟，无效时 `-1` |
| `getArrivalInTicks()` | `vibration` 的到达时间，无效时 `-1` |
| `getItem()` | `item` 粒子的 [ItemStackHelper](inventory.md)，无效时 `null` |
| `getBlock()` | `block`/`falling_dust` 等粒子的 [BlockStateHelper](blocks.md)，无效时 `null` |

## 实体 {#entity}

实体进出渲染范围、受伤回血、玩家进出服务器列表。典型示例——苦力怕雷达：

```javascript
JsMacros.on("EntityLoad", JavaWrapper.methodToJava((e) => {
    if (e.entity.getType() === "minecraft:creeper") {
        Chat.log("§c附近出现苦力怕！")
    }
}))
```

### EntityLoad

实体进入客户端渲染范围时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `entity` | [EntityHelper](entities.md) | 加载的实体 |

### EntityUnload

实体离开客户端渲染范围（或死亡等）时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `entity` | [EntityHelper](entities.md) | 卸载的实体 |
| `reason` | `EntityUnloadReason`（字符串） | 卸载原因 |

### EntityDamaged

任意实体受伤时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `entity` | [EntityHelper](entities.md) | 受伤的实体 |
| `health` | `number` | 受伤后的血量 |
| `damage` | `number` | 伤害量 |

### EntityHealed

任意实体回血时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `entity` | [EntityHelper](entities.md) | 回血的实体 |
| `health` | `number` | 恢复后的血量 |
| `damage` | `number` | 恢复量（字段名沿用 `damage`） |

### NameChange

实体显示名（自定义名牌）变化时触发。**可取消**：是 · **适合 joined**：是（可改 `newName`）

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `entity` | [EntityHelper](entities.md) | 相关实体 |
| `oldName` | [TextHelper](chat.md) 或 `null` | 旧名字 |
| `newName` | [TextHelper](chat.md) 或 `null`（**可写**） | 新名字，joined 中可替换 |

### PlayerJoin

玩家出现在 Tab 玩家列表时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `UUID` | `string` | 玩家 UUID |
| `player` | [PlayerListEntryHelper](entities.md) | 玩家列表条目 |

### PlayerLeave

玩家从 Tab 玩家列表消失时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `UUID` | `string` | 玩家 UUID |
| `player` | [PlayerListEntryHelper](entities.md) | 玩家列表条目 |

## 界面 {#screen}

屏幕开关、Boss 栏、告示牌编辑。典型示例——记录打开的界面：

```javascript
JsMacros.on("OpenScreen", JavaWrapper.methodToJava((e) => {
    Chat.log(`打开界面: ${e.screenName}`)
}))
```

### OpenScreen

打开（或关闭到）任意界面时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `screen` | [IScreen](screen.md) 或 `null` | 界面对象，关闭界面时为 `null` |
| `screenName` | `ScreenName`（字符串） | 界面名，如 `Survival Inventory`、`Chat`、`3 Row Chest`（见 [常用类型](types.md)） |

### Bossbar

Boss 栏增删/更新时触发。**可取消**：否 · **适合 joined**：否

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `bossBar` | `BossBarHelper` 或 `null` | Boss 栏对象（`REMOVE` 时可能为 `null`） |
| `uuid` | `string` | Boss 栏 UUID |
| `type` | `'ADD'`/`'REMOVE'`/`'UPDATE_PERCENT'`/`'UPDATE_NAME'`/`'UPDATE_STYLE'`/`'UPDATE_PROPERTIES'` | 更新类型 |

### SignEdit

打开告示牌编辑界面前触发。**可取消**：是 · **适合 joined**：是（可预填文本、可跳过界面）

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `pos` | [Pos3D](position.md) | 告示牌位置 |
| `closeScreen` | `boolean`（**可写**） | 设为 `true` 直接提交而不打开编辑界面 |
| `front` | `boolean` | 编辑的是否为正面 |
| `signText` | `JavaList<string>` 或 `null`（**可写**） | 四行文本，joined 中可预填 |

## 网络 {#network}

数据包级监听，功能强大但**量非常大**，务必配合过滤器使用，详见 [数据包](packets.md) 与 [网络](network.md)。典型示例——只监听特定包：

```javascript
const filter = JsMacros.eventFilters().compile("RecvPacket", 'eq(event.type, "BlockUpdateS2CPacket")')
JsMacros.on("RecvPacket", filter, JavaWrapper.methodToJava((e) => {
    Chat.log("收到一个方块更新包")
}))
```

### RecvPacket

客户端收到任意数据包时触发。**可取消**：是 · **适合 joined**：是（可取消、可替换 `packet`）

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `packet` | 原始包对象（**可写**，可为 `null`） | Minecraft 数据包，joined 中可替换 |
| `type` | `PacketName`（字符串） | 包类型名，如 `BlockUpdateS2CPacket` |

| 方法 | 说明 |
| --- | --- |
| `getPacketBuffer()` | 返回 [PacketByteBufferHelper](packets.md)，读改包数据；改完用 `toPacket()` 生成新包替换 `packet` |

### SendPacket

客户端发送任意数据包前触发。**可取消**：是 · **适合 joined**：是（可取消、可替换 `packet`）

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `packet` | 原始包对象（**可写**，可为 `null`） | 即将发送的数据包 |
| `type` | `PacketName`（字符串） | 包类型名 |

| 方法 | 说明 |
| --- | --- |
| `replacePacket(...args)` | 用给定构造参数替换为同类型新包（`packet` 为 `null` 时抛异常；推荐优先用 `getPacketBuffer()`） |
| `getPacketBuffer()` | 返回 [PacketByteBufferHelper](packets.md) |

## 脚本与其他 {#script}

自定义事件、服务生命周期、自定义命令与编辑器扩展。典型示例——脚本间通信：

```javascript
// 发送方
const ev = JsMacros.createCustomEvent("myScripts:ping")
ev.putString("msg", "hello")
ev.trigger()

// 接收方（另一个脚本）
JsMacros.on("myScripts:ping", JavaWrapper.methodToJava((e) => {
    Chat.log(`收到: ${e.getString("msg")}`)
}))
```

### Custom

自定义事件，由 `JsMacros.createCustomEvent(name)` 创建并 `trigger()` 触发。**可取消**：由 `cancelable` 标志决定 · **适合 joined**：由 `joinable` 标志决定

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `eventName` | `string` | 事件名 |
| `joinable` | `boolean`（**可写**） | 是否可 join |
| `cancelable` | `boolean`（**可写**） | 是否可取消 |

方法：`trigger()`、`triggerAsync(callback)`、`putInt`/`putString`/`putDouble`/`putBoolean`/`putObject`、`getInt`/`getString`/`getDouble`/`getBoolean`/`getObject`、`getType(name)`、`getUnderlyingMap()`、`joinable()`、`cancellable()`、`registerEvent()`。完整表格与示例见 [事件系统 · 自定义事件](events.md)。

### Service

服务脚本启动时作为全局 `event` 传入（不是用 `on` 监听的事件）。**可取消**：否 · **适合 joined**：—

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `serviceName` | `string` | 服务名 |
| `stopListener` | `MethodWrapper` 或 `null`（**可写**） | 服务停止时执行 |
| `postStopListener` | `MethodWrapper` 或 `null`（**可写**） | 停止流程最后执行 |

| 方法 | 说明 |
| --- | --- |
| `unregisterOnStop(offEvents, ...registrables)` | 停止时自动清理：`stopListener` → 注销事件监听 → unregister 传入的 Draw2D/Draw3D/CommandBuilder 等 → `postStopListener` |
| `putInt` / `putString` / `putDouble` / `putBoolean` / `putObject` | 向**全局变量空间**写值（跨脚本共享） |
| `getInt` / `getString` / `getDouble` / `getBoolean` / `getObject` / `getType` | 从全局变量空间读值 |
| `getAndIncrementInt` / `getAndDecrementInt` / `incrementAndGetInt` / `decrementAndGetInt` | 整数原子自增/自减 |
| `toggleBoolean` | 翻转布尔值并返回新值 |
| `remove(key)` / `getRaw()` | 删除键 / 获取底层 `JavaMap` |

详见 [服务](services.md)。

### CommandContext

用 CommandBuilder 注册的自定义命令被执行时触发。**可取消**：否 · **适合 joined**：—

| 方法 | 说明 |
| --- | --- |
| `getArg(name)` | 取命令参数（参数不存在会抛 `CommandSyntaxException`） |
| `getInput()` | 完整命令输入串 |
| `getRaw()` | 原始 Brigadier `CommandContext` |
| `getChild()` | 子上下文 |
| `getRange()` | 输入范围 `StringRange` |

### WrappedScript

由 `JsMacros.wrapScriptRun` / `wrapScriptRunAsync` 包装的脚本被当作函数调用时触发（作为该脚本的全局 `event`）。**可取消**：否 · **适合 joined**：—

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `arg1` | 泛型 | 第一个调用参数 |
| `arg2` | 泛型 | 第二个调用参数 |
| `result` | 泛型（**可写**） | 返回值 |

| 方法 | 说明 |
| --- | --- |
| `setReturnBoolean(b)` / `setReturnInt(i)` / `setReturnDouble(d)` / `setReturnString(s)` / `setReturnObject(o)` | 按类型设置返回值 |

### CodeRender

JsMacros 内置脚本编辑器渲染代码时触发，用于自定义语法高亮/自动补全主题。**可取消**：否 · **适合 joined**：—

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `cursor` | `SelectCursor` | 编辑器光标 |
| `code` | `string` | 当前代码文本 |
| `language` | `string` | 代码语言 |
| `screen` | `EditorScreen` | 编辑器界面 |
| `textLines` | `JavaList<`[TextHelper](chat.md)`>` | 需要你填充的着色文本行（不填则不渲染） |
| `autoCompleteSuggestions` | `JavaList<AutoCompleteSuggestion>` | 需要你填充的补全建议 |
| `rightClickActions` | `MethodWrapper`（**可写**） | 右键菜单动作生成器：`(index) => Map<名称, 动作>` |

| 方法 | 说明 |
| --- | --- |
| `genPrismNodes()` | 生成 Prism4j 语法节点树 |
| `createTextBuilder()` | 新建 TextBuilder（配合 `textLines`） |
| `createSuggestion(startIndex, suggestion)` / `createSuggestion(startIndex, suggestion, displayText)` | 创建补全建议 |
| `createMap()` | 创建 `LinkedHashMap`（配合 `rightClickActions`） |
| `createPrefixTree()` | 创建 `StringHashTrie` 前缀树（大数据量补全加速） |
| `getThemeData()` | 主题颜色表 `Map<string, number[]>` |

---

!!! tip "没找到想要的事件？"
    以上就是 2.1.0 的全部 59 个事件（`interface Events` 名单一个不落）。旧版教程里的 `JoinedTick`、`ScreenChange` 等事件在本版本中不存在，请用 `Tick`、`OpenScreen` 替代。事件机制与监听技巧回到 [事件系统](events.md)。
