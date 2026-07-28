---
icon: lucide/globe-2
---

# 世界与方块

`World` 是读取当前世界的全局对象：方块、实体、玩家列表、记分板、Boss 栏、Tab 列表、区块、声音粒子、服务器状态都从这里入手。它只负责"读世界"，方块自身的详细数据见[方块 Helper](blocks.md)，实体操作见[实体](entities.md)。

!!! warning "先判空是所有世界脚本的第一行"
    绝大多数方法在**世界尚未加载**时返回 `null`（数值类方法返回 `-1`）。养成习惯：

    ```javascript
    if (!World.isWorldLoaded()) {
        Chat.log("还没进入世界")
        // 事件脚本里可以直接 return
    }
    ```

## 世界状态

### 加载与标识

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isWorldLoaded()` | `boolean` | 是否已加载世界 |
| `getWorldIdentifier()` | `string` | 基于存档名/服务器 IP 的世界标识，取不到时为 `"UNKNOWN_NAME"`。适合做"每个世界一份配置"的键 |
| `getDimension()` | `string` 或 `null` | 当前维度，如 `"minecraft:overworld"`。部分服务器会用自定义维度 ID 区分子世界 |
| `getRaw()` | 原始对象 或 `null` | 原版 `ClientLevel` 对象，反射进阶用 |

### 时间与天气

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getTime()` | `number` | 世界创建以来经过的刻数，未加载时 `-1` |
| `getTimeOfDay()` | `number` | 一天内的时间刻数（**包含**睡觉跳过的刻），未加载时 `-1` |
| `isDay()` / `isNight()` | `boolean` | 是否白天 / 夜晚 |
| `isRaining()` / `isThundering()` | `boolean` | 是否下雨 / 打雷 |
| `getMoonPhase()` | `number` | 月相（`0`~`7`，`0` 为满月），未加载时 `-1` |

!!! tip "把 getTimeOfDay 换算成游戏时刻"
    `getTimeOfDay() % 24000` 才是当天时间：`0` 日出、`6000` 正午、`13000` 入夜、`18000` 午夜。

```javascript
const t = World.getTimeOfDay() % 24000
if (t >= 12500) {
    Chat.log("天要黑了，快回家")
}
```

### 环境与光照

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getBiome()` | `string` 或 `null` | 玩家所在群系，如 `"minecraft:plains"` |
| `getBiomeAt(x, z)` | `string` 或 `null` | 指定坐标的群系（区块须已加载） |
| `getBiomeAt(x, y, z)` | `string` 或 `null` | 带 y 的版本，洞穴群系需要它 |
| `getDifficulty()` | `number` | 难度 `0`~`3`（和平~困难），未加载时 `-1` |
| `getRespawnPos()` | `BlockPosHelper` 或 `null` | 重生点 |
| `getSkyLight(x, y, z)` | `number` | 天空光照 `0`~`15`，未加载时 `-1` |
| `getBlockLight(x, y, z)` | `number` | 方块光照 `0`~`15`，未加载时 `-1` |

```javascript
const p = Player.getPlayer()
if (World.isWorldLoaded() && p) {
    const pos = p.getBlockPos()
    Chat.log(`维度: ${World.getDimension()} 群系: ${World.getBiome()}`)
    Chat.log(`脚下光照: 方块 ${World.getBlockLight(pos.getX(), pos.getY(), pos.getZ())}`
        + ` / 天空 ${World.getSkyLight(pos.getX(), pos.getY(), pos.getZ())}`)
}
```

## 读取方块

`getBlock` 返回 [`BlockDataHelper`](blocks.md)（方块 + 状态 + NBT 的组合），拿到之后的玩法都在[方块 Helper](blocks.md)页。

| 重载 | 说明 |
| --- | --- |
| `getBlock(x, y, z)` | 整数坐标 |
| `getBlock(pos)` | 接收 `Pos3D` |
| `getBlock(blockPos)` | 接收 `BlockPosHelper` |

区块未加载或世界未加载时返回 `null`。

```javascript
const block = World.getBlock(0, 64, 0)
if (block) {
    Chat.log(`${block.getId()} ${block.getName().getString()}`)
}
```

### 遍历一片区域

回调会对区域内每个方块调用一次，记得用 `JavaWrapper.methodToJava` 包装：

| 方法 | 说明 |
| --- | --- |
| `iterateSphere(pos, radius, callback)` | 以 `pos` 为球心遍历，默认跳过空气 |
| `iterateSphere(pos, radius, ignoreAir, callback)` | `ignoreAir = false` 时空气也回调 |
| `iterateBox(pos1, pos2, callback)` | 遍历两点围成的长方体 |
| `iterateBox(pos1, pos2, ignoreAir, callback)` | 同上，可包含空气 |

```javascript
const center = Player.getPlayer().getBlockPos()
World.iterateSphere(center, 4, JavaWrapper.methodToJava((block) => {
    if (block.getId() === "minecraft:chest") {
        Chat.log(`箱子: ${block.getX()}, ${block.getY()}, ${block.getZ()}`)
    }
}))
```

!!! note "大范围遍历用扫描器"
    `iterateSphere`/`iterateBox` 每个方块都要跨语言回调一次，范围大了会很慢。批量搜索请用下文的 [findBlocksMatching](#findblocksmatching) 或[世界扫描器](world_scanner.md)。

## 获取实体

| 重载 | 说明 |
| --- | --- |
| `getEntities()` | 渲染距离内全部实体 |
| `getEntities(...types)` | 按类型筛选，如 `"zombie"`、`"minecraft:creeper"`（可省略 `minecraft:` 前缀，可传多个） |
| `getEntities(distance)` | 距玩家 `distance` 格以内的实体 |
| `getEntities(distance, ...types)` | 距离 + 类型 |
| `getEntities(filter)` | 自定义回调过滤，返回 `true` 保留 |

返回 `JavaList<EntityHelper>` 或 `null`。

```javascript
// 8 格内的僵尸
const zombies = World.getEntities(8, "zombie")
if (zombies && zombies.size() > 0) {
    Chat.log(`最近的僵尸: ${zombies.get(0).getName().getString()}`)
}

// 自定义过滤：所有掉落物
const items = World.getEntities(JavaWrapper.methodToJava((e) => e.getType() === "minecraft:item"))
if (items) Chat.log(`附近有 ${items.size()} 个掉落物`)
```

实体的坐标、血量、追踪等方法见[实体](entities.md)。

### 射线检测

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `rayTraceBlock(x1, y1, z1, x2, y2, z2, fluid)` | `BlockDataHelper` 或 `null` | 两点之间打射线，返回命中的第一个方块；`fluid` 为是否命中流体 |
| `rayTraceEntity(x1, y1, z1, x2, y2, z2)` | `EntityHelper` 或 `null` | 两点之间命中的第一个实体 |

```javascript
const player = Player.getPlayer()
if (player) {
    const eye = player.getEyePos()
    const look = Player.createLookingVector(player).scale(5)
    const end = eye.add(look.getDeltaX(), look.getDeltaY(), look.getDeltaZ())
    const block = World.rayTraceBlock(eye.getX(), eye.getY(), eye.getZ(), end.getX(), end.getY(), end.getZ(), false)
    if (block) Chat.log(`看向的方块: ${block.getId()}`)
}
```

!!! tip
    只是想知道准心指着什么，直接用 `Player.rayTraceBlock(distance, fluid)` / `Player.rayTraceEntity()` 更省事。

## 玩家列表

两个"玩家列表"含义不同，别混用：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getLoadedPlayers()` | `JavaList<PlayerEntityHelper>` 或 `null` | **渲染距离内**的玩家实体，有坐标、血量等实体数据 |
| `getPlayers()` | `JavaList<PlayerListEntryHelper>` 或 `null` | **Tab 列表**上的全部玩家，有延迟、皮肤、队伍等条目数据 |
| `getPlayerEntry(name)` | `PlayerListEntryHelper` 或 `null` | 按名字查 Tab 列表条目 |

```javascript
// 附近真实加载的玩家（能拿到实体数据）
const nearby = World.getLoadedPlayers()
if (nearby) {
    for (const p of nearby) {
        Chat.log(`${p.getPlayerName()} 血量 ${p.getHealth()}`)
    }
}

// 整个服务器在线的玩家（Tab 列表）
const entry = World.getPlayerEntry("Steve")
if (entry) {
    Chat.log(`${entry.getName()} 延迟 ${entry.getPing()}ms 模式 ${entry.getGamemode()}`)
}
```

## Tab 列表

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getTabListHeader()` | `TextHelper` 或 `null` | Tab 列表顶部文字 |
| `getTabListFooter()` | `TextHelper` 或 `null` | Tab 列表底部文字 |

### PlayerListEntryHelper 方法表

`getPlayers()` / `getPlayerEntry(name)` 返回的条目对象：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getUUID()` | `string` 或 `null` | 玩家 UUID |
| `getName()` | `string` 或 `null` | 玩家名 |
| `getDisplayText()` | `TextHelper` | Tab 里的显示文本（含前后缀/颜色） |
| `getPing()` | `number` | 网络延迟（毫秒） |
| `getGamemode()` | `string` | 游戏模式，未知时为 `null` |
| `getTeam()` | `TeamHelper` 或 `null` | 所在队伍（见下文记分板） |
| `hasCape()` | `boolean` | 是否启用披风 |
| `hasSlimModel()` | `boolean` | 是否 Alex 细手臂模型 |
| `getSkinTexture()` | `string` | 皮肤纹理标识，未知时 `null` |
| `getSkinUrl()` | `string` 或 `null` | 皮肤纹理 URL |
| `getCapeTexture()` / `getCapeUrl()` | `string` 或 `null` | 披风纹理标识 / URL |
| `getElytraTexture()` / `getElytraUrl()` | `string` 或 `null` | 鞘翅纹理标识 / URL |
| `getPublicKey()` | `byte[]` | 聊天签名公钥 |

```javascript
const header = World.getTabListHeader()
if (header) Chat.log(`Tab 标题: ${header.getString()}`)

const players = World.getPlayers()
if (players) {
    for (const e of players) {
        Chat.log(`${e.getDisplayText().getString()} (${e.getPing()}ms)`)
    }
}
```

## 记分板

服务器脚本的刚需：侧边栏（sidebar）上的金币、任务进度、队伍信息几乎都靠它读。入口只有一个：

```javascript
const scoreboards = World.getScoreboards()   // ScoreboardsHelper 或 null
```

### ScoreboardsHelper 方法表

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getCurrentScoreboard()` | `ScoreboardObjectiveHelper` 或 `null` | **当前显示的侧边栏**，最常用 |
| `getObjectiveSlot(slot)` | `ScoreboardObjectiveHelper` 或 `null` | 按槽位取：`0` Tab 列表、`1` 侧边栏、`2` 名字下方、`3 + 颜色序号` 各队伍专属侧边栏（最大 `18`） |
| `getObjectiveForTeamColorIndex(index)` | `ScoreboardObjectiveHelper` 或 `null` | 按队伍颜色序号取侧边栏 |
| `getTeams()` | `JavaList<TeamHelper>` | 全部队伍 |
| `getPlayerTeam()` | `TeamHelper` | 自己所在队伍 |
| `getPlayerTeam(player)` | `TeamHelper` | 指定玩家所在队伍 |
| `getPlayerTeamColorIndex()` | `number` | 自己队伍的颜色序号 |
| `getPlayerTeamColorIndex(player)` | `number` | 指定玩家队伍的颜色序号 |
| `getTeamColor()` / `getTeamColor(player)` | `number` | 队伍颜色值，无队伍时 `-1` |
| `getTeamColorName()` / `getTeamColorName(player)` | `string` 或 `null` | 队伍颜色名 |
| `getTeamColorFormatting()` / `getTeamColorFormatting(player)` | `FormattingHelper` 或 `null` | 队伍颜色的格式化对象 |

### ScoreboardObjectiveHelper 方法表

一个"目标"（objective）就是一块记分板，比如侧边栏：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getName()` | `string` | 目标内部名 |
| `getDisplayName()` | `TextHelper` | 显示标题 |
| `getTexts()` | `JavaList<TextHelper>` | **按显示顺序**的每一行文本，读侧边栏首选 |
| `getPlayerScores()` | `JavaMap<string, number>` | 条目名 → 分数 |
| `scoreToDisplayName()` | `JavaMap<number, TextHelper>` | 分数 → 显示文本 |
| `getKnownPlayers()` | `JavaList<string>` | 所有条目名 |
| `getKnownPlayersDisplayNames()` | `JavaList<TextHelper>` | 所有条目的显示文本 |

### TeamHelper 方法表

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getName()` | `string` | 队伍内部名 |
| `getDisplayName()` | `TextHelper` | 队伍显示名 |
| `getPlayerList()` | `JavaList<string>` | 队伍成员名单 |
| `getPrefix()` / `getSuffix()` | `TextHelper` | 成员名字的前缀 / 后缀 |
| `getColorIndex()` | `number` | 颜色序号（`getColor()` 已废弃，用这个） |
| `getColorValue()` | `number` | RGB 颜色值，无颜色时 `-1` |
| `getColorName()` | `string` | 颜色名，如 `"red"` |
| `getColorFormat()` | `FormattingHelper` | 颜色格式化对象 |
| `getScoreboard()` | `ScoreboardsHelper` | 所属记分板 |
| `getCollisionRule()` | `string` | 碰撞规则 |
| `isFriendlyFire()` | `boolean` | 是否允许友伤 |
| `showFriendlyInvisibles()` | `boolean` | 能否看见隐身队友 |
| `nametagVisibility()` | `string` | 名牌可见性规则 |
| `deathMessageVisibility()` | `string` | 死亡消息可见性规则 |

### 示例：读出整个侧边栏

```javascript
const scoreboards = World.getScoreboards()
if (!scoreboards) {
    Chat.log("世界未加载")
} else {
    const sidebar = scoreboards.getCurrentScoreboard()
    if (!sidebar) {
        Chat.log("当前没有侧边栏")
    } else {
        Chat.log(`== ${sidebar.getDisplayName().getString()} ==`)
        for (const line of sidebar.getTexts()) {
            Chat.log(line.getString())
        }
    }
}
```

!!! tip "读金币/等级这类数字"
    很多服务器把数据写在行文本里而不是分数里，先 `getTexts()` 打印一遍，再用正则从对应行提取：
    `const gold = line.getString().match(/金币[:：]\s*(\d+)/)`。

## Boss 栏

```javascript
const bars = World.getBossBars()   // JavaMap<string, BossBarHelper>，键是 UUID
```

### BossBarHelper 方法表

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getUUID()` | `string` | Boss 栏 UUID |
| `getName()` | `TextHelper` | 标题文本 |
| `getPercent()` | `number` | 剩余百分比（`0.0`~`1.0`） |
| `getColor()` | `string` | 颜色，如 `"purple"` |
| `getColorValue()` | `number` | 颜色数值 |
| `getColorFormat()` | `FormattingHelper` | 颜色格式化对象 |
| `getStyle()` | `string` | 分段样式（notch 风格） |

```javascript
const bars = World.getBossBars()
for (const uuid of bars.keySet()) {
    const bar = bars.get(uuid)
    Chat.log(`${bar.getName().getString()} 剩余 ${Math.round(bar.getPercent() * 100)}%`)
}
```

!!! note
    不少服务器拿 Boss 栏当公告栏/倒计时用，`getName().getString()` 配合正则同样可以提取信息。

## 服务器信息

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getCurrentServerAddress()` | `string` 或 `null` | 当前服务器地址（`地址/IP:端口` 格式），单人时为 `null` |
| `getServerTPS()` | `string` | 一句话汇总：`即时TPS, 1M: x, 5M: x, 15M: x` |
| `getServerInstantTPS()` | `number` | 即时 TPS 估算 |
| `getServer1MAverageTPS()` | `number` | 近 1 分钟平均 TPS |
| `getServer5MAverageTPS()` | `number` | 近 5 分钟平均 TPS |
| `getServer15MAverageTPS()` | `number` | 近 15 分钟平均 TPS |

TPS 是客户端根据时间同步包估算的，仅供参考。

### ServerInfoHelper

想拿服务器 MOTD、在线人数、版本这类"服务器列表信息"，用的是 [`Client`](client.md) 命名空间的 `Client.ping(ip)` / `Client.pingAsync(ip, callback)`，它们返回 `ServerInfoHelper`：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getName()` | `string` | 服务器名（多人列表里的名字） |
| `getAddress()` | `string` | 地址 |
| `getLabel()` | `TextHelper` | MOTD |
| `getPlayerCountLabel()` | `TextHelper` | 在线人数文本 |
| `getPlayerListSummary()` | `JavaList<TextHelper>` | 悬停可见的玩家摘要 |
| `getPing()` | `number` | 延迟 |
| `getVersion()` | `TextHelper` | 版本文本 |
| `getProtocolVersion()` | `number` | 协议版本号 |
| `resourcePackPolicy()` | `string` | 资源包策略 |
| `getIcon()` | `byte[]` | 服务器图标数据 |
| `isOnline()` / `isLocal()` | `boolean` | 是否在线 / 是否局域网 |
| `getNbt()` | `NBTCompoundHelper` | 原始 NBT |

```javascript
const info = Client.ping("mc.hypixel.net")
Chat.log(`${info.getName()} | ${info.getPlayerCountLabel().getString()} | ${info.getPing()}ms`)
```

## 声音与粒子

### playSound 重载

| 重载 | 说明 |
| --- | --- |
| `playSound(id)` | 默认音量播放 |
| `playSound(id, volume)` | 指定音量（`1.0` 为原始音量） |
| `playSound(id, volume, pitch)` | 指定音量和音调（`0.5`~`2.0`） |
| `playSound(id, volume, pitch, x, y, z)` | 在世界坐标处播放（有 3D 定位感） |

```javascript
World.playSound("minecraft:block.note_block.pling", 1, 2)

const p = Player.getPlayer().getPos()
World.playSound("minecraft:entity.experience_orb.pickup", 1, 1, p.getX(), p.getY(), p.getZ())
```

`playSoundFile(file, volume)` 可以播放脚本目录下的音频文件（返回 `javax.sound` 的 `Clip` 对象），适合自定义提示音。

### spawnParticle 重载

只在本地客户端显示，别人看不到：

| 重载 | 说明 |
| --- | --- |
| `spawnParticle(id, x, y, z, count)` | 在坐标处生成 `count` 个粒子 |
| `spawnParticle(id, x, y, z, deltaX, deltaY, deltaZ, speed, count, force)` | 带扩散范围和速度；`force = true` 时超过 32 格也显示 |

```javascript
const p = Player.getPlayer().getPos()
World.spawnParticle("minecraft:heart", p.getX(), p.getY() + 2, p.getZ(), 5)
World.spawnParticle("minecraft:flame", p.getX(), p.getY() + 1, p.getZ(), 0.5, 0.5, 0.5, 0.01, 30, true)
```

## 区块

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getChunk(x, z)` | `ChunkHelper` 或 `null` | 注意参数是**区块坐标**：`x >> 4`、`z >> 4` |
| `isChunkLoaded(chunkX, chunkZ)` | `boolean` | 区块是否在渲染距离内且已加载 |

### ChunkHelper 方法表

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getChunkX()` / `getChunkZ()` | `number` | 区块坐标（不是世界坐标） |
| `getStartingBlock()` | `BlockPosHelper` | 区块起始角（本地 0,0,0 对应的世界坐标） |
| `getOffsetBlock(xOffset, y, zOffset)` | `BlockPosHelper` | 起始角偏移后的位置（`xOffset`/`zOffset` 为 `0`~`15`，`y` 是真实高度） |
| `getMinBuildHeight()` / `getMaxBuildHeight()` | `number` | 最低 / 最高建筑高度 |
| `getHeight()` | `number` | 区块总高度 |
| `getTopYAt(xOffset, zOffset, heightmap)` | `number` | 指定高度图下该列的最高 y |
| `getBiome(xOffset, y, zOffset)` | `string` | 区块内某点的群系 |
| `getInhabitedTime()` | `number` | 玩家累计停留时间（影响局部难度） |
| `getEntities()` | `JavaList<EntityHelper>` | 区块内全部实体 |
| `getTileEntities()` | `JavaList<BlockPosHelper>` | 区块内全部方块实体（箱子、熔炉等）的位置 |
| `forEach(includeAir, callback)` | `ChunkHelper` | 遍历区块内每个方块，回调收到 `BlockDataHelper` |
| `containsAny(...blocks)` | `boolean` | 区块内是否存在任一指定方块 |
| `containsAll(...blocks)` | `boolean` | 区块内是否同时存在全部指定方块 |
| `getHeightmaps()` | 集合 | 原始高度图数据 |
| `getSurfaceHeightmap()` / `getOceanFloorHeightmap()` | 原始对象 | 地表 / 海底高度图 |
| `getMotionBlockingHeightmap()` / `getMotionBlockingNoLeavesHeightmap()` | 原始对象 | 阻挡移动的高度图（含/不含树叶） |

```javascript
const p = Player.getPlayer().getBlockPos()
const chunk = World.getChunk(p.getX() >> 4, p.getZ() >> 4)
if (chunk) {
    Chat.log(`当前区块 (${chunk.getChunkX()}, ${chunk.getChunkZ()})`)
    Chat.log(`方块实体数量: ${chunk.getTileEntities().size()}`)
    if (chunk.containsAny("diamond_ore", "deepslate_diamond_ore")) {
        Chat.log("这个区块里有钻石!")
    }
}
```

## 搜索方块：findBlocksMatching {#findblocksmatching}

在玩家周围若干**区块**内批量找方块，返回 `JavaList<Pos3D>` 或 `null`。挖矿、找刷怪笼、找箱子的首选入门方案。

| 重载 | 说明 |
| --- | --- |
| `findBlocksMatching(id, chunkrange)` | 以玩家所在区块为中心，找一种方块 |
| `findBlocksMatching(ids, chunkrange)` | 找多种方块，`ids` 是数组 |
| `findBlocksMatching(centerX, centerZ, id, chunkrange)` | 指定中心**区块坐标**，找一种方块 |
| `findBlocksMatching(centerX, centerZ, ids, chunkrange)` | 指定中心区块坐标，找多种方块 |
| `findBlocksMatching(blockFilter, stateFilter, chunkrange)` | 回调过滤：`blockFilter` 收到 `BlockHelper`，`stateFilter` 收到 `BlockStateHelper`（可传 `null`），都返回 `boolean` |
| `findBlocksMatching(chunkX, chunkZ, blockFilter, stateFilter, chunkrange)` | 回调过滤 + 指定中心区块 |

方块 ID 可省略 `minecraft:` 前缀。

```javascript
// 找钻石矿（普通 + 深板岩两种）
const blocks = World.findBlocksMatching(["diamond_ore", "deepslate_diamond_ore"], 2)
if (blocks) {
    Chat.log(`附近找到 ${blocks.size()} 个钻石矿`)
    for (const pos of blocks) {
        Chat.log(`  -> ${pos.x}, ${pos.y}, ${pos.z}`)
    }
}
```

回调重载可以按任意条件过滤，例如"所有需要工具挖掘的发光方块"：

```javascript
const found = World.findBlocksMatching(
    JavaWrapper.methodToJava((block) => block.getId().includes("chest")),
    null,   // 不过滤方块状态
    1
)
if (found) Chat.log(`找到 ${found.size()} 个含 chest 的方块`)
```

!!! warning "chunkrange 不是格数，性能按平方增长"
    `chunkrange = 1` 表示以中心区块为准的 3×3 区块（48×48 格全高度），`chunkrange = n` 要扫 `(2n+1)²` 个区块。从 `1`~`2` 开始试，别一上来就 `10`。回调重载因为每个候选方块都要跨语言调用，比 ID 重载慢得多——复杂条件请改用[世界扫描器](world_scanner.md)。

## 更强的搜索：WorldScanner

`findBlocksMatching` 每次调用都是一锤子买卖；而 `World.getWorldScanner()` 可以先把过滤条件"编译"成一个可复用的**扫描器**，支持链式组合条件、Java 侧并行扫描、按球体/立方体/触及范围等多种区域扫描，还能统计区块内方块数量。找矿 ESP、自动农场这类需要反复扫描的脚本都应该用它：

```javascript
const scanner = World.getWorldScanner()
    .withStringBlockFilter().contains("diamond_ore")
    .build()
Chat.log(`附近钻石矿: ${scanner.scanAroundPlayer(2).size()} 个`)
```

完整用法（Builder 全部方法、扫描区域重载、性能对比、ESP 实战）见[世界扫描器](world_scanner.md)。
