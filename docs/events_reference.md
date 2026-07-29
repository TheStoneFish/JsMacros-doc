---
icon: lucide/list
---

# 全部事件参考

事件机制（on/off、joined、waitForEvent、自定义事件）见 [事件系统](events.md)。

!!! note "阅读须知"
    - 参数列标 `*` 的字段**可写**（joined 监听中修改才生效）；参数为空表示该事件无字段（只能用 `getEventName()`）。
    - 字段类型见各事件说明的简名，`JavaList<T>` 用 `size()`/`get(i)` 访问（见 [常用类型](types.md)）。
    - `PacketName`、`SoundId`、`ScreenName`、`Dimension` 等类型实际是字符串标识，直接当字符串比较即可。

| 名称 | 真实名称 | 参数 | 方法 |
| --- | --- | --- | --- |
| 启动游戏 | `LaunchGame` | `playerName` | — |
| 档案加载 | `ProfileLoad` | `profileName` | — |
| 加入服务器 | `JoinServer` | `player`、`address` | — |
| 断开连接 | `Disconnect` | `message` | — |
| 退出游戏 | `QuitGame` | — | — |
| 游戏刻 | `Tick` | — | — |
| 维度切换 | `DimensionChange` | `dimension` | — |
| 资源包加载 | `ResourcePackLoaded` | `isGameStart`、`loadedPacks` | — |
| 发送消息 | `SendMessage` | `message *` | — |
| 接收消息 | `RecvMessage` | `text *`、`signature`、`messageType` | — |
| 标题显示 | `Title` | `type`、`message *` | — |
| 血量变化 | `HealthChange` | `health`、`change` | — |
| 受伤 | `Damage` | `attacker`、`source`、`health`、`change` | — |
| 回血 | `Heal` | `source`、`health`、`change` | — |
| 死亡 | `Death` | `deathPos`、`inventory` | `respawn()` |
| 饥饿变化 | `HungerChange` | `foodLevel` | — |
| 氧气变化 | `AirChange` | `air` | — |
| 经验变化 | `EXPChange` | `progress`、`total`、`level`、`prevProgress`、`prevTotal`、`prevLevel` | — |
| 盔甲变化 | `ArmorChange` | `slot`、`item`、`oldItem` | — |
| 药水效果更新 | `StatusEffectUpdate` | `oldEffect`、`newEffect`、`added`、`removed` | — |
| 鞘翅滑翔 | `FallFlying` | `state` | — |
| 骑乘 | `Riding` | `state`、`entity` | — |
| 按键 | `Key` | `action`、`key`、`mods` | — |
| 鼠标滚轮 | `MouseScroll` | `deltaX`、`deltaY` | — |
| 交互方块 | `InteractBlock` | `offhand`、`result`、`block`、`side` | — |
| 交互实体 | `InteractEntity` | `offhand`、`result`、`entity` | — |
| 攻击方块 | `AttackBlock` | `block`、`side` | — |
| 攻击实体 | `AttackEntity` | `entity` | — |
| 打开容器 | `OpenContainer` | `inventory`、`screen` | — |
| 容器更新 | `ContainerUpdate` | `inventory`、`screen` | — |
| 点击槽位 | `ClickSlot` | `mode`、`button`、`slot` | `getInventory()` |
| 槽位更新 | `SlotUpdate` | `type`、`slot`、`oldStack`、`newStack` | `getInventory()` |
| 丢出槽位 | `DropSlot` | `slot`、`all` | `getInventory()` |
| 手持切换 | `HeldItemChange` | `offHand`、`item`、`oldItem` | — |
| 捡起物品 | `ItemPickup` | `item` | — |
| 物品损耗 | `ItemDamage` | `item`、`damage` | — |
| 区块加载 | `ChunkLoad` | `x`、`z`、`isFull` | — |
| 区块卸载 | `ChunkUnload` | `x`、`z` | — |
| 方块更新 | `BlockUpdate` | `block`、`updateType` | — |
| 声音 | `Sound` | `sound`、`volume`、`pitch`、`position` | — |
| 服务器声音 | `ServerSound` | `sound *`、`source *`、`position *`、`entity *`、`volume *`、`pitch *`、`seed *` | `getSeed()`、`setSeed()` |
| 粒子 | `Particle` | `raw`、`type`、`x`/`y`/`z`、`offsetX`/`offsetY`/`offsetZ`、`speed`、`count`、`longDistance` | `getPos()`、`is()`、`getColor()`、`getHexColor()`、`getScale()`、`getItem()`、`getBlock()` 等 |
| 实体加载 | `EntityLoad` | `entity` | — |
| 实体卸载 | `EntityUnload` | `entity`、`reason` | — |
| 实体受伤 | `EntityDamaged` | `entity`、`health`、`damage` | — |
| 实体回血 | `EntityHealed` | `entity`、`health`、`damage` | — |
| 名字变化 | `NameChange` | `entity`、`oldName`、`newName *` | — |
| 玩家加入列表 | `PlayerJoin` | `UUID`、`player` | — |
| 玩家离开列表 | `PlayerLeave` | `UUID`、`player` | — |
| 打开界面 | `OpenScreen` | `screen`、`screenName` | — |
| Boss 栏 | `Bossbar` | `bossBar`、`uuid`、`type` | — |
| 告示牌编辑 | `SignEdit` | `pos`、`closeScreen *`、`front`、`signText *` | — |
| 接收数据包 | `RecvPacket` | `packet *`、`type` | `getPacketBuffer()` |
| 发送数据包 | `SendPacket` | `packet *`、`type` | `replacePacket()`、`getPacketBuffer()` |
| 自定义事件 | `Custom` | `eventName`、`joinable *`、`cancelable *` | `trigger()`、`putXxx`/`getXxx`、`registerEvent()` 等 |
| 服务 | `Service` | `serviceName`、`stopListener *`、`postStopListener *` | `unregisterOnStop()`、`putXxx`/`getXxx`、`toggleBoolean` 等 |
| 命令上下文 | `CommandContext` | — | `getArg()`、`getInput()`、`getRaw()`、`getChild()`、`getRange()` |
| 包装脚本 | `WrappedScript` | `arg1`、`arg2`、`result *` | `setReturnBoolean()`、`setReturnInt()`、`setReturnString()` 等 |
| 代码渲染 | `CodeRender` | `cursor`、`code`、`language`、`screen`、`textLines`、`autoCompleteSuggestions`、`rightClickActions *` | `genPrismNodes()`、`createTextBuilder()`、`createSuggestion()`、`getThemeData()` 等 |

!!! tip "没找到想要的事件？"
     你可以查看官方文档是否有这个事件。事件机制与监听技巧回到 [事件系统](events.md)。
