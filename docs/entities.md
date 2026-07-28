---
icon: lucide/users
---

# 实体

`EntityHelper` 是所有实体 helper 的基类。玩家、动物、怪物、掉落物等会有更具体的子类，但常用读取都在基类和 `LivingEntityHelper` 里。

继承关系大致如下（只列本页涉及的部分）：

```
EntityHelper                    所有实体的基类
├── ItemEntityHelper            掉落物
└── LivingEntityHelper          有血量的生物
    ├── MobEntityHelper         有 AI 的生物（怪物、动物）
    ├── PlayerEntityHelper      玩家
    └── MerchantEntityHelper    可交易实体（流浪商人等）
        └── VillagerEntityHelper 村民
```

更细分的实体类（苦力怕、狼、盔甲架、鱼漂……）见[特化实体参考](entities_reference.md)。

## 实体从哪来

### World.getEntities 各种重载

| 调用方式 | 返回内容 |
| --- | --- |
| `World.getEntities()` | 渲染距离内的全部实体 |
| `World.getEntities(...types)` | 渲染距离内匹配指定类型的实体 |
| `World.getEntities(distance)` | 距玩家 `distance` 格以内的实体 |
| `World.getEntities(distance, ...types)` | 距离 + 类型双重过滤 |
| `World.getEntities(filter)` | 用回调函数自定义过滤 |

所有重载都返回 `JavaList<EntityHelper>` 或 `null`（未进入世界时），使用前先判空：

```javascript
const entities = World.getEntities(6)
if (!entities) return

for (const entity of entities) {
    Chat.log(`${entity.getType()} ${entity.distanceTo(Player.getPlayer())}`)
}
```

按类型过滤（类型 ID 可以省略 `minecraft:` 前缀，可传多个）：

```javascript
const zombies = World.getEntities(8, "zombie")
const undead = World.getEntities(16, "zombie", "husk", "skeleton")
```

用回调自定义过滤（回调返回真值则保留）：

```javascript
const lowHealthMobs = World.getEntities(JavaWrapper.methodToJava(e =>
    e.is("zombie", "creeper") && e.asLiving().getHealth() < 10
))
```

获取所有已加载玩家用 `World.getLoadedPlayers()`，返回 `PlayerEntityHelper` 列表。

### 事件字段

很多事件直接带 `entity` 字段，不用自己搜索：

| 事件 | 相关字段 | 触发时机 |
| --- | --- | --- |
| `AttackEntity` | `entity` | 玩家攻击实体 |
| `InteractEntity` | `entity`、`offhand`、`result` | 玩家右键实体 |
| `EntityLoad` | `entity` | 实体进入渲染范围 |
| `EntityUnload` | `entity`、`reason` | 实体卸载 |
| `EntityDamaged` | `entity`、`health`、`damage` | 实体受伤 |
| `EntityHealed` | `entity`、`health`、`damage` | 实体恢复 |

```javascript
JsMacros.on("AttackEntity", JavaWrapper.methodToJava((event) => {
    Chat.log(`攻击了 ${event.entity.getName().getString()}`)
}))
```

### 射线（准星指向的实体）

| 调用方式 | 说明 |
| --- | --- |
| `Player.rayTraceEntity()` | 准星指向的实体（默认距离） |
| `Player.rayTraceEntity(distance)` | 限定距离（整数） |
| `World.rayTraceEntity(x1, y1, z1, x2, y2, z2)` | 两点之间命中的第一个实体 |
| `entity.rayTraceEntity(distance)` | 从任意实体的视线方向做射线 |
| `entity.rayTraceBlock(distance, fluid)` | 从实体视线方向找方块，返回 [BlockDataHelper](blocks.md) |

```javascript
const target = Player.rayTraceEntity(5)
if (target) Chat.log(`正在看着：${target.getName().getString()}`)
```

## EntityHelper 通用方法

### 位置与运动

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getPos()` | `Pos3D` | 实体坐标，见[位置](position.md) |
| `getBlockPos()` | `BlockPosHelper` | 所在方块坐标 |
| `getEyePos()` | `Pos3D` | 眼睛位置 |
| `getX()` / `getY()` / `getZ()` | `number` | 单轴坐标 |
| `getEyeHeight()` | `number` | 当前眼高偏移 |
| `getYaw()` / `getPitch()` | `number` | 水平/垂直朝向 |
| `getVelocity()` | `Pos3D` | 速度向量 |
| `getSpeed()` | `number` | 当前速度（格/秒） |
| `getFacingDirection()` | `DirectionHelper` | 面朝方向（取整到 45 度） |
| `getChunkPos()` | `Pos2D` | 区块坐标（注意 z 存在 y 字段里） |
| `getChunk()` | `ChunkHelper` | 所在区块，见[世界](world.md) |
| `getBiome()` | `string` | 所在生物群系 ID |

### 身份信息

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getName()` | `TextHelper` | 显示名，`getString()` 转字符串 |
| `getType()` | `string` | 实体类型 ID，如 `minecraft:zombie` |
| `getUUID()` | `string` | UUID（非玩家实体是随机的） |
| `getNBT()` | `NBTCompoundHelper` | NBT 数据 |
| `is(...types)` | `boolean` | 类型是否匹配任一给定 ID |

### 状态判断

```javascript
if (entity.isAlive() && entity.isReallyAlive()) {
    Chat.log("实体还活着")
}
```

| 方法 | 作用 |
| --- | --- |
| `isAlive()` | 实体是否存活 |
| `isReallyAlive()` | 2.1.0 新增：存活且和玩家在同一世界，更适合过滤异常/卸载状态 |
| `isGlowing()` | 是否发光 |
| `isInLava()` | 是否在岩浆 |
| `isOnFire()` | 是否着火 |
| `isSneaking()` / `isSprinting()` | 潜行/疾跑 |
| `getAir()` / `getMaxAir()` | 当前/最大氧气值 |

### 载具与乘客

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getVehicle()` | `EntityHelper` 或 `null` | 骑乘的载具 |
| `getPassengers()` | `JavaList<EntityHelper>` 或 `null` | 背上的乘客 |

### 距离和视线

`distanceTo` 有四种重载：

| 调用方式 | 说明 |
| --- | --- |
| `distanceTo(entity)` | 到另一实体的距离 |
| `distanceTo(blockPos)` | 到 `BlockPosHelper` 的距离 |
| `distanceTo(pos3d)` | 到 `Pos3D` 的距离 |
| `distanceTo(x, y, z)` | 到坐标的距离 |

```javascript
const player = Player.getPlayer()
const zombies = World.getEntities(8, "zombie")
if (!zombies || zombies.isEmpty()) return
const nearest = zombies.get(0)

Chat.log(nearest.distanceTo(player))
Chat.log(player.canSeeEntity(nearest))  // canSeeEntity 在 LivingEntityHelper 上
```

实体可对方块/实体做射线：

```javascript
const block = player.rayTraceBlock(5, false)
const target = player.rayTraceEntity(5)
```

### 客户端修改（视觉）

| 方法 | 说明 |
| --- | --- |
| `setCustomName(text)` | 设置自定义名（`TextHelper` 或 `null` 清除） |
| `setCustomNameVisible(b)` | 名字是否常显 |
| `setGlowing(val)` | 设置发光 |
| `resetGlowing()` | 恢复真实发光状态 |
| `setGlowingColor(color)` | 设置发光颜色（RGB 整数） |
| `resetGlowingColor()` | 恢复默认发光颜色 |
| `getGlowingColor()` | 当前发光颜色（受 setGlowingColor 影响） |

## 类型判断与转换（is → as 模式）

`is(...)` 检查实体类型是否匹配任一给定 ID（可省略 `minecraft:` 前缀）：

```javascript
entity.is("creeper")            // 是不是苦力怕
entity.is("zombie", "husk")     // 任一匹配即为 true
```

`as` 系列做类型转换：

| 方法 | 目标 |
| --- | --- |
| `asClientPlayer()` | 本地玩家（`ClientPlayerEntityHelper`） |
| `asPlayer()` | 玩家（`PlayerEntityHelper`） |
| `asLiving()` | 生物（`LivingEntityHelper`） |
| `asAnimal()` | 动物（按 d.ts 返回 `LivingEntityHelper`） |
| `asItem()` | 掉落物（`ItemEntityHelper`） |
| `asVillager()` | 村民（`VillagerEntityHelper`） |
| `asMerchant()` | 可交易实体（`MerchantEntityHelper`） |
| `asServerEntity()` | 服务端实体 helper，只在单人/局域网有效，可能为 `null` |

!!! tip "JsMacros 返回的实体本身就是正确子类"
    `World.getEntities()` 等 API 内部会自动创建正确的子类 helper（比如苦力怕拿到的就是 `CreeperEntityHelper`）。`as` 系列主要是给 TypeScript 补类型用的，在 JavaScript 里 `is()` 判断通过后可以直接调用子类方法。但转换前不判断类型，调用子类方法会直接报错，所以标准写法永远是先 `is` 再用。

遍历实体并安全转换的完整示例：

```javascript
const entities = World.getEntities(16)
if (!entities) return

for (const e of entities) {
    if (e.is("item")) {
        // 掉落物：读取包含的物品
        const stack = e.asItem().getContainedItemStack()
        Chat.log(`掉落物：${stack.getName().getString()} x${stack.getCount()}`)
    } else if (e.is("villager")) {
        // 村民：读职业和等级
        const v = e.asVillager()
        Chat.log(`村民：${v.getProfession()} Lv.${v.getLevel()}`)
    } else if (e.is("creeper")) {
        // 特化类型：is() 通过后可直接调用子类方法
        Chat.log(`苦力怕，充能：${e.isCharged()}`)
    } else if (e.getType().includes("zombie")) {
        // 生物通用：转 LivingEntityHelper 读血量
        const living = e.asLiving()
        Chat.log(`${living.getName().getString()} ${living.getHealth()}/${living.getMaxHealth()}`)
    }
}
```

## 生物实体 LivingEntityHelper

生物、玩家、怪物通常能转成 `LivingEntityHelper`。

```javascript
const living = entity.asLiving()
Chat.log(`${living.getHealth()} / ${living.getMaxHealth()}`)
```

### 生命与防御

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getHealth()` / `getMaxHealth()` | `number` | 当前/最大血量 |
| `getAbsorptionHealth()` | `number` | 伤害吸收（金心） |
| `getDefaultHealth()` | `number` | 该实体类型的默认血量 |
| `getArmor()` | `number` | 护甲值 |
| `isReallyAliveAndHealthy()` | `boolean` | 2.1.0 推荐健康判断：存活、同世界且血量大于 0 |
| `canTakeDamage()` | `boolean` | 是否能受到伤害 |

### 装备与手持

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getMainHand()` / `getOffHand()` | `ItemStackHelper` | 主副手[物品](items.md) |
| `getHeadArmor()` / `getChestArmor()` / `getLegArmor()` / `getFootArmor()` | `ItemStackHelper` | 头/胸/腿/脚装备 |
| `isHolding(itemId)` | `boolean` | 任一手是否拿着指定物品 |
| `getBowPullProgress()` | `number` | 拉弓进度（默认 0） |
| `getItemUseTimeLeft()` | `number` | 物品使用剩余时间 |

### 状态效果

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getStatusEffects()` | `JavaList<StatusEffectHelper>` | 全部状态效果 |
| `hasStatusEffect(id)` | `boolean` | 是否有某状态效果 |
| `canHaveStatusEffect(effect)` | `boolean` | 是否可以拥有某效果 |

!!! warning "状态效果不同步到客户端"
    除了玩家自己，其他实体的状态效果服务器一般不会同步给客户端，`hasStatusEffect` 对别的实体很可能永远返回 `false`。读自己的效果用 `Player.getPlayer().getStatusEffects()` 最可靠。

### 姿态与其他判断

| 方法 | 说明 |
| --- | --- |
| `isSleeping()` | 是否在床上 |
| `isFallFlying()` | 是否鞘翅飞行 |
| `isOnGround()` | 是否在地面 |
| `isBaby()` | 是否幼年 |
| `isUndead()` | 是否亡灵生物 |
| `isSpectator()` | 是否旁观模式 |
| `isPartOfGame()` | 存活且非旁观 |
| `canBreatheInWater()` | 能否水下呼吸 |
| `hasNoDrag()` / `hasNoGravity()` | 无阻力/无重力 |
| `canTarget(living)` | 能否把某生物作为目标 |
| `canSeeEntity(entity)` | 是否有到目标的视线 |
| `canSeeEntity(entity, simpleCast)` | 同上，可选简化射线 |
| `getMobTags()` | 实体身上的标签列表 |

### MobEntityHelper

有 AI 的生物（怪物和大部分动物）额外提供：

| 方法 | 说明 |
| --- | --- |
| `isAttacking()` | 是否正在攻击 |
| `isAiDisabled()` | AI 是否被禁用（禁用后不移动不攻击） |

## 状态效果 StatusEffectHelper

`getStatusEffects()` 返回的每个元素是 `StatusEffectHelper`：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getId()` | `string` | 效果 ID，如 `minecraft:speed` |
| `getStrength()` | `number` | 等级（0 = I 级） |
| `getTime()` | `number` | 剩余时间（tick） |
| `getCategory()` | `string` | `"BENEFICIAL"`、`"NEUTRAL"` 或 `"HARMFUL"` |
| `isPermanent()` | `boolean` | 是否永久 |
| `isAmbient()` | `boolean` | 是否信标类环境效果（粒子更透明） |
| `hasIcon()` | `boolean` | 是否有图标 |
| `isVisible()` | `boolean` | 是否显示粒子 |
| `isInstant()` | `boolean` | 是否瞬间效果（如瞬间伤害） |
| `isBeneficial()` / `isNeutral()` / `isHarmful()` | `boolean` | 分类快捷判断 |

```javascript
const effects = Player.getPlayer().getStatusEffects()
for (const eff of effects) {
    const secs = Math.floor(eff.getTime() / 20)
    Chat.log(`${eff.getId()} ${eff.getStrength() + 1} 级，剩余 ${secs} 秒 (${eff.getCategory()})`)
}
```

## 掉落物 ItemEntityHelper

掉落在地上的物品是 `item` 类型实体，`getContainedItemStack()` 返回包含的[物品](items.md)：

```javascript
const drops = World.getEntities(16, "item")
if (!drops) return

for (const drop of drops) {
    const stack = drop.asItem().getContainedItemStack()
    Chat.log(`${stack.getName().getString()} x${stack.getCount()}`)
}
```

## 村民与交易

### MerchantEntityHelper

村民和流浪商人都属于可交易实体：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getTrades()` | `JavaList<TradeOfferHelper>` | 交易列表 |
| `refreshTrades()` | `JavaList<TradeOfferHelper>` | 刷新并返回交易列表 |
| `getExperience()` | `number` | 商人经验 |
| `hasCustomer()` | `boolean` | 是否正在和人交易 |

!!! warning "交易数据依赖服务端"
    官方 JSDoc 明确提示：`getTrades()` 等方法取决于服务器发来的数据，可能只在单人游戏可用。多人服务器上一般需要先打开交易界面才能拿到交易列表。

### VillagerEntityHelper

村民在商人基础上额外提供：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getProfession()` | `string` | 职业 ID，如 `minecraft:farmer` |
| `getStyle()` | `string` | 外观风格（生物群系样式） |
| `getLevel()` | `number` | 交易等级（1 新手 ~ 5 大师） |

### TradeOfferHelper 交易条目

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getInput()` | `JavaList<ItemStackHelper>` | 全部输入[物品](items.md) |
| `getLeftInput()` | `ItemStackHelper` | 第一个输入（数量已含涨价/折扣） |
| `getRightInput()` | `ItemStackHelper` | 第二个输入（可能为空物品） |
| `getOutput()` | `ItemStackHelper` | 产出物品 |
| `getIndex()` | `number` | 在交易界面里的序号 |
| `select()` | `TradeOfferHelper` | 在打开的交易界面上选中此交易 |
| `isAvailable()` | `boolean` | 是否可交易（未锁定） |
| `getUses()` / `getMaxUses()` | `number` | 已用次数 / 锁定前最大次数 |
| `getExperience()` | `number` | 交易给玩家的经验 |
| `shouldRewardPlayerExperience()` | `boolean` | 成交后是否掉经验球 |
| `getOriginalFirstInput()` | `ItemStackHelper` | 未调价的原始第一输入 |
| `getOriginalPrice()` | `number` | 原价 |
| `getAdjustedPrice()` | `number` | 调整后的实际价格 |
| `getCurrentPriceAdjustment()` | `number` | 当前价格调整量，负数是折扣 |
| `getSpecialPrice()` | `number` | 声望/村庄英雄折扣，负数便宜 |
| `getPriceMultiplier()` | `number` | 价格波动系数（普通 0.05，护甲工具 0.2） |
| `getDemandBonus()` | `number` | 需求加价（补货时结算，最低 0） |
| `getNBT()` | `NBTCompoundHelper` | 交易的 NBT 数据 |

完整示例——列出最近村民的交易并找出打折条目：

```javascript
// 建议在单人游戏中、站在村民旁边运行
const villagers = World.getEntities(5, "villager")
if (!villagers || villagers.isEmpty()) {
    Chat.log("附近没有村民")
} else {
    const villager = villagers.get(0).asVillager()
    Chat.log(`职业：${villager.getProfession()}  等级：${villager.getLevel()}`)

    const trades = villager.getTrades()
    for (const trade of trades) {
        const inputs = trade.getInput()
        const cost = []
        for (const item of inputs) {
            cost.push(`${item.getName().getString()} x${item.getCount()}`)
        }
        const out = trade.getOutput()
        const state = trade.isAvailable() ? "" : "（已锁定）"
        const discount = trade.getCurrentPriceAdjustment() < 0 ? " [打折中]" : ""
        Chat.log(`${cost.join(" + ")} -> ${out.getName().getString()} x${out.getCount()}${state}${discount}`)
    }
    // 如果已经打开了这个村民的交易界面，可以直接选中某笔交易：
    // trades.get(0).select()
}
```

## 发光高亮（ESP）

`setGlowing` / `setGlowingColor` 只改本地渲染，常用来做实体高亮：

```javascript
// 高亮 30 格内所有苦力怕为红色，10 秒后恢复
const creepers = World.getEntities(30, "creeper")
if (creepers) {
    for (const c of creepers) {
        c.setGlowingColor(0xFF0000).setGlowing(true)
    }
    Client.waitTick(200)
    for (const c of creepers) {
        c.resetGlowingColor()
        c.resetGlowing()
    }
}
```

想要持续高亮，把上面的逻辑放进[服务](services.md)里，每隔若干 tick 重新扫描一次即可。

```javascript
entity.setGlowing(true)
entity.setGlowingColor(0x00FF00)
Client.waitTick(100)
entity.resetGlowingColor()
entity.resetGlowing()
```

!!! warning "客户端视觉效果"
    发光设置主要影响本地显示。多人服务器里不要把它当作服务端状态。

## PlayerEntityHelper

玩家实体在 `LivingEntityHelper` 基础上额外提供（更多玩家操作见[玩家](player.md)）：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getPlayerName()` | `string` | 玩家真实名字（非显示名） |
| `getXP()` | `number` | 总经验 |
| `getXPLevel()` | `number` | 经验等级 |
| `getXPProgress()` | `number` | 当前级进度（0~1） |
| `getXPToLevelUp()` | `number` | 升级还需经验 |
| `getAttackCooldownProgress()` | `number` | 攻击冷却进度 |
| `getAttackCooldownProgressPerTick()` | `number` | 每 tick 冷却增量 |
| `getAbilities()` | `PlayerAbilitiesHelper` | 飞行、创造、速度等能力 |
| `getFishingBobber()` | `FishingBobberEntityHelper` 或 `null` | 鱼漂实体，没钓鱼时为 `null` |
| `isSleeping()` | `boolean` | 是否在睡觉 |
| `isSleepingLongEnough()` | `boolean` | 是否睡够跳过夜晚的时长 |
| `getScore()` | `number` | 玩家分数 |

```javascript
const players = World.getLoadedPlayers()
if (players) {
    for (const p of players) {
        Chat.log(`${p.getPlayerName()} Lv.${p.getXPLevel()}`)
    }
}
```

---

想查某种具体生物（苦力怕引信、狼的驯服状态、盔甲架姿势……）有哪些专属方法，请看[特化实体参考](entities_reference.md)。
