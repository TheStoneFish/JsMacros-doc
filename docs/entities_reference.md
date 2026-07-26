---
icon: lucide/paw-print
---

# 特化实体参考

除了[实体基础](entities.md)里的通用方法，JsMacros 还为几十种具体实体提供了特化 Helper。`World.getEntities()` 等 API 返回的对象本身就是正确的子类：拿到的苦力怕就是 `CreeperEntityHelper`，可以直接调用专属方法；不确定类型时先用 `is()` 判断。

```javascript
// 方式一：按类型过滤，拿到的直接就是特化 Helper
const creepers = World.getEntities(20, "creeper")

// 方式二：遍历全部实体，用 is() 判断后再调用特化方法
for (const e of World.getEntities(20)) {
    if (e.is("creeper") && e.isIgnited()) Chat.log("有苦力怕被点燃了！")
}
```

!!! note "没有 asCreeper() 这类方法"
    `EntityHelper` 上只有 `asLiving()`、`asVillager()` 等少数几个通用转换。特化类型不需要转换——返回值本来就是子类，`is()` 通过后直接用即可。

## 总索引：实体类型 → Helper 类

| 分组 | 实体 | 类型 ID | Helper 类 |
| --- | --- | --- | --- |
| Boss | 末影龙 | `ender_dragon` | `EnderDragonEntityHelper` |
| Boss | 凋灵 | `wither` | `WitherEntityHelper` |
| 装饰 | 盔甲架 | `armor_stand` | `ArmorStandEntityHelper` |
| 装饰 | 末地水晶 | `end_crystal` | `EndCrystalEntityHelper` |
| 装饰 | 物品展示框 | `item_frame` / `glow_item_frame` | `ItemFrameEntityHelper` |
| 装饰 | 画 | `painting` | `PaintingEntityHelper` |
| 展示 | 方块展示实体 | `block_display` | `BlockDisplayEntityHelper` |
| 展示 | 物品展示实体 | `item_display` | `ItemDisplayEntityHelper` |
| 展示 | 文本展示实体 | `text_display` | `TextDisplayEntityHelper` |
| 敌对 | 猪灵（含蛮兵，基类） | `piglin` / `piglin_brute` | `AbstractPiglinEntityHelper` |
| 敌对 | 烈焰人 | `blaze` | `BlazeEntityHelper` |
| 敌对 | 苦力怕 | `creeper` | `CreeperEntityHelper` |
| 敌对 | 溺尸 | `drowned` | `DrownedEntityHelper` |
| 敌对 | 末影人 | `enderman` | `EndermanEntityHelper` |
| 敌对 | 恶魂 | `ghast` | `GhastEntityHelper` |
| 敌对 | 守卫者 | `guardian` / `elder_guardian` | `GuardianEntityHelper` |
| 敌对 | 灾厄村民（基类） | 掠夺者/卫道士/唤魔者等 | `IllagerEntityHelper` |
| 敌对 | 幻翼 | `phantom` | `PhantomEntityHelper` |
| 敌对 | 猪灵 | `piglin` | `PiglinEntityHelper` |
| 敌对 | 掠夺者 | `pillager` | `PillagerEntityHelper` |
| 敌对 | 潜影贝 | `shulker` | `ShulkerEntityHelper` |
| 敌对 | 史莱姆 | `slime` | `SlimeEntityHelper` |
| 敌对 | 施法灾厄村民 | `evoker` / `illusioner` | `SpellcastingIllagerEntityHelper` |
| 敌对 | 蜘蛛 | `spider` | `SpiderEntityHelper` |
| 敌对 | 恼鬼 | `vex` | `VexEntityHelper` |
| 敌对 | 卫道士 | `vindicator` | `VindicatorEntityHelper` |
| 敌对 | 监守者 | `warden` | `WardenEntityHelper` |
| 敌对 | 女巫 | `witch` | `WitchEntityHelper` |
| 敌对 | 僵尸（基类） | `zombie` 等 | `ZombieEntityHelper` |
| 敌对 | 僵尸村民 | `zombie_villager` | `ZombieVillagerEntityHelper` |
| 其他 | 效果云（滞留药水） | `area_effect_cloud` | `AreaEffectCloudEntityHelper` |
| 其他 | 下落方块 | `falling_block` | `FallingBlockEntityHelper` |
| 其他 | 交互实体 | `interaction` | `InteractionEntityHelper` |
| 其他 | 点燃的 TNT | `tnt` | `TntEntityHelper` |
| 友好 | 动物（基类） | — | `AnimalEntityHelper` |
| 友好 | 可驯服动物（基类） | — | `TameableEntityHelper` |
| 友好 | 马类（基类） | — | `AbstractHorseEntityHelper` |
| 友好 | 马 | `horse` | `HorseEntityHelper` |
| 友好 | 驴/带箱马类 | `donkey` / `mule` | `DonkeyEntityHelper` |
| 友好 | 羊驼 | `llama` / `trader_llama` | `LlamaEntityHelper` |
| 友好 | 悦灵 | `allay` | `AllayEntityHelper` |
| 友好 | 美西螈 | `axolotl` | `AxolotlEntityHelper` |
| 友好 | 蝙蝠 | `bat` | `BatEntityHelper` |
| 友好 | 蜜蜂 | `bee` | `BeeEntityHelper` |
| 友好 | 猫 | `cat` | `CatEntityHelper` |
| 友好 | 海豚 | `dolphin` | `DolphinEntityHelper` |
| 友好 | 鱼类（基类） | `cod` / `salmon` 等 | `FishEntityHelper` |
| 友好 | 狐狸 | `fox` | `FoxEntityHelper` |
| 友好 | 青蛙 | `frog` | `FrogEntityHelper` |
| 友好 | 山羊 | `goat` | `GoatEntityHelper` |
| 友好 | 铁傀儡 | `iron_golem` | `IronGolemEntityHelper` |
| 友好 | 哞菇 | `mooshroom` | `MooshroomEntityHelper` |
| 友好 | 豹猫 | `ocelot` | `OcelotEntityHelper` |
| 友好 | 熊猫 | `panda` | `PandaEntityHelper` |
| 友好 | 鹦鹉 | `parrot` | `ParrotEntityHelper` |
| 友好 | 猪 | `pig` | `PigEntityHelper` |
| 友好 | 北极熊 | `polar_bear` | `PolarBearEntityHelper` |
| 友好 | 河豚 | `pufferfish` | `PufferfishEntityHelper` |
| 友好 | 兔子 | `rabbit` | `RabbitEntityHelper` |
| 友好 | 绵羊 | `sheep` | `SheepEntityHelper` |
| 友好 | 雪傀儡 | `snow_golem` | `SnowGolemEntityHelper` |
| 友好 | 炽足兽 | `strider` | `StriderEntityHelper` |
| 友好 | 热带鱼 | `tropical_fish` | `TropicalFishEntityHelper` |
| 友好 | 狼 | `wolf` | `WolfEntityHelper` |
| 弹射物 | 箭 | `arrow` 等 | `ArrowEntityHelper` |
| 弹射物 | 鱼漂 | `fishing_bobber` | `FishingBobberEntityHelper` |
| 弹射物 | 三叉戟 | `trident` | `TridentEntityHelper` |
| 弹射物 | 凋灵之首 | `wither_skull` | `WitherSkullEntityHelper` |
| 载具 | 船 | `boat` / `chest_boat` 等 | `BoatEntityHelper` |
| 载具 | 动力矿车 | `furnace_minecart` | `FurnaceMinecartEntityHelper` |
| 载具 | TNT 矿车 | `tnt_minecart` | `TntMinecartEntityHelper` |

## Boss

### 末影龙 EnderDragonEntityHelper

继承 `MobEntityHelper`。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getPhase()` | `string` | 当前阶段：`HoldingPattern`、`StrafePlayer`、`LandingApproach`、`Landing`、`Takeoff`、`SittingFlaming`、`SittingScanning`、`SittingAttacking`、`ChargingPlayer`、`Dying`、`Hover` |
| `getBodyPart(index)` | `EntityHelper` | 按序号取身体部位 |
| `getBodyParts()` | `JavaList<EntityHelper>` | 全部身体部位 |
| `getBodyParts(name)` | `JavaList<EntityHelper>` | 按名称取部位：`head`、`neck`、`body`、`tail`、`wing` |

### 凋灵 WitherEntityHelper

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getRemainingInvulnerableTime()` | `number` | 剩余无敌时间（tick） |
| `isInvulnerable()` | `boolean` | 是否无敌（刚召唤默认 220 tick） |
| `isFirstPhase()` | `boolean` | 是否第一阶段 |
| `isSecondPhase()` | `boolean` | 是否第二阶段（免疫弹射物并向玩家俯冲） |

## 装饰实体

### 盔甲架 ArmorStandEntityHelper

继承 `LivingEntityHelper`，所以装备读取直接用 `getHeadArmor()` 等方法。旋转值格式统一为 `[yaw, pitch, roll]` 数组。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isVisible()` | `boolean` | 是否可见 |
| `isSmall()` | `boolean` | 是否小型 |
| `hasArms()` | `boolean` | 是否有手臂 |
| `hasBasePlate()` | `boolean` | 是否有底座 |
| `isMarker()` | `boolean` | 是否 marker（无碰撞箱） |
| `getHeadRotation()` | `number[]` | 头部旋转 |
| `getBodyRotation()` | `number[]` | 身体旋转 |
| `getLeftArmRotation()` / `getRightArmRotation()` | `number[]` | 左/右臂旋转 |
| `getLeftLegRotation()` / `getRightLegRotation()` | `number[]` | 左/右腿旋转 |

```javascript
// 读取最近盔甲架的姿势和装备
const stands = World.getEntities(8, "armor_stand")
if (stands && !stands.isEmpty()) {
    const stand = stands.get(0)
    const [yaw, pitch, roll] = stand.getHeadRotation()
    Chat.log(`头部姿势 yaw=${yaw} pitch=${pitch} roll=${roll}`)
    Chat.log(`头上戴着：${stand.getHeadArmor().getName().getString()}`)
}
```

### 末地水晶 / 物品展示框 / 画

| 类 | 方法 | 返回 | 说明 |
| --- | --- | --- | --- |
| `EndCrystalEntityHelper` | `isNatural()` | `boolean` | 是否自然生成（自然生成的有基岩底座） |
| | `getBeamTarget()` | `BlockPosHelper` 或 `null` | 光束指向的[位置](position.md) |
| `ItemFrameEntityHelper` | `isGlowingFrame()` | `boolean` | 是否荧光展示框 |
| | `getRotation()` | `number` | 框内物品的旋转档位 |
| | `getItem()` | `ItemStackHelper` | 框内[物品](items.md) |
| `PaintingEntityHelper` | `getWidth()` / `getHeight()` | `number` | 画的宽/高 |
| | `getIdentifier()` | `string` | 画作 ID |

## 展示实体（display）

1.19.4+ 的展示实体，三种类型共用基类 `DisplayEntityHelper`：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getLerpTargetX()` / `getLerpTargetY()` / `getLerpTargetZ()` | `number` | 插值目标坐标 |
| `getLerpTargetYaw()` / `getLerpTargetPitch()` | `number` | 插值目标朝向 |
| `getLerpProgress(delta)` | `number` | 插值进度 |
| `getVisibilityBoundingBox()` | `Vec3D` | 可见性包围盒 |
| `getBillboardMode()` | `string` | `"fixed"`、`"vertical"`、`"horizontal"` 或 `"center"` |
| `getBrightness()` / `getSkyBrightness()` / `getBlockBrightness()` | `number` | 亮度信息 |
| `getViewRange()` | `number` | 可视距离 |
| `getShadowRadius()` / `getShadowStrength()` | `number` | 阴影半径/强度 |
| `getDisplayWidth()` / `getDisplayHeight()` | `number` | 显示尺寸 |
| `getGlowColorOverride()` | `number` | 发光颜色覆盖值 |

各子类专属方法：

| 类 | 方法 | 返回 | 说明 |
| --- | --- | --- | --- |
| `BlockDisplayEntityHelper` | `getBlockState()` | `BlockStateHelper` 或 `null` | 展示的[方块](blocks.md)状态 |
| `ItemDisplayEntityHelper` | `getItem()` | `ItemStackHelper` | 展示的[物品](items.md) |
| | `getTransform()` | `string` 或 `null` | 展示模式：`none`、`thirdperson_lefthand`、`thirdperson_righthand`、`firstperson_lefthand`、`firstperson_righthand`、`head`、`gui`、`ground`、`fixed` |
| `TextDisplayEntityHelper` | `getData()` | `TextDisplayDataHelper` 或 `null` | 文本渲染数据 |

`TextDisplayDataHelper`（`getData()` 的返回值）：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getText()` | `TextHelper` | 显示的文本 |
| `getLineWidth()` | `number` | 行宽 |
| `getTextOpacity()` | `number` | 文字不透明度 |
| `getBackgroundColor()` | `number` | 背景颜色 |
| `hasShadowFlag()` | `boolean` | 是否有阴影 |
| `hasSeeThroughFlag()` | `boolean` | 是否可透视 |
| `hasDefaultBackgroundFlag()` | `boolean` | 是否默认背景 |
| `getAlignment()` | `string` | 对齐：`"center"`、`"left"` 或 `"right"` |

## 敌对生物（mob）

### 苦力怕 CreeperEntityHelper

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isCharged()` | `boolean` | 是否闪电充能 |
| `isIgnited()` | `boolean` | 是否已被点燃 |
| `getFuseChange()` | `number` | 引信每 tick 变化量：正在充能为正、正在解除为负 |
| `getFuseTime()` | `number` | 已充能时间 |
| `getMaxFuseTime()` | `number` | 爆炸前最大充能时间 |
| `getRemainingFuseTime()` | `number` | 距爆炸剩余时间，没在充能时为 `-1` |

```javascript
// 苦力怕临爆报警
const creepers = World.getEntities(10, "creeper")
if (creepers) {
    for (const c of creepers) {
        if (c.getRemainingFuseTime() >= 0) {
            Chat.actionbar(`苦力怕要炸了！剩余 ${c.getRemainingFuseTime()} tick`)
        }
    }
}
```

### 末影人 EndermanEntityHelper

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isScreaming()` | `boolean` | 是否处于尖叫状态 |
| `isProvoked()` | `boolean` | 是否被玩家激怒 |
| `isHoldingBlock()` | `boolean` | 是否搬着方块 |
| `getHeldBlock()` | `BlockStateHelper` 或 `null` | 搬的[方块](blocks.md) |

### 守卫者 GuardianEntityHelper

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isElder()` | `boolean` | 是否远古守卫者 |
| `hasTarget()` | `boolean` | 是否有攻击目标 |
| `getTarget()` | `EntityHelper` 或 `null` | 激光指向的目标 |
| `hasSpikesRetracted()` | `boolean` | 尖刺是否伸出 |

### 猪灵 AbstractPiglinEntityHelper / PiglinEntityHelper

`AbstractPiglinEntityHelper`（猪灵和猪灵蛮兵共同基类）只有 `canBeZombified()`——当前维度是否会僵尸化。`PiglinEntityHelper` 额外提供：

| 方法 | 说明 |
| --- | --- |
| `isWandering()` | 是否在闲逛 |
| `isDancing()` | 是否随音乐跳舞 |
| `isAdmiring()` | 是否在端详物品（以物易物中） |
| `isMeleeAttacking()` | 是否在近战攻击 |
| `isChargingCrossbow()` | 是否在装填弩 |
| `hasCrossbowReady()` | 弩是否已装填完毕 |

### 灾厄村民 IllagerEntityHelper 系列

`IllagerEntityHelper` 是掠夺者、卫道士、唤魔者、幻术师的基类：

| 类 | 方法 | 返回 | 说明 |
| --- | --- | --- | --- |
| `IllagerEntityHelper` | `isCelebrating()` | `boolean` | 是否在庆祝（袭击胜利） |
| | `getState()` | `string` | 当前行为状态 |
| `SpellcastingIllagerEntityHelper` | `isCastingSpell()` | `boolean` | 是否正在施法 |
| | `getCastedSpell()` | `string` | 正在施放的法术 |
| `PillagerEntityHelper` | `isCaptain()` | `boolean` | 是否袭击队长（举着灾厄旗帜） |
| `VindicatorEntityHelper` | `isJohnny()` | `boolean` | 是否 Johnny（攻击一切生物） |

### 监守者 WardenEntityHelper

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getAnger()` | `number` | 对当前目标的愤怒值 |
| `isDigging()` | `boolean` | 是否在钻入地下 |
| `isEmerging()` | `boolean` | 是否在破土而出 |
| `isRoaring()` | `boolean` | 是否在咆哮 |
| `isSniffing()` | `boolean` | 是否在嗅探 |
| `isChargingSonicBoom()` | `boolean` | 是否在蓄力音波攻击 |

### 僵尸家族与其他

| 类 | 方法 | 返回 | 说明 |
| --- | --- | --- | --- |
| `ZombieEntityHelper` | `isConvertingToDrowned()` | `boolean` | 是否正在转化成溺尸 |
| `ZombieVillagerEntityHelper` | `isConvertingToVillager()` | `boolean` | 是否正在被治疗回村民 |
| | `getVillagerBiomeType()` | `string` | 被感染前的村民群系样式 |
| | `getProfession()` | `string` | 被感染前的职业 |
| | `getLevel()` | `number` | 被感染前的交易等级 |
| `DrownedEntityHelper` | `hasTrident()` | `boolean` | 是否拿着三叉戟 |
| | `hasNautilusShell()` | `boolean` | 是否拿着鹦鹉螺壳 |
| `WitchEntityHelper` | `isDrinkingPotion()` | `boolean` | 是否正在喝药水 |
| | `getPotion()` | `ItemStackHelper` | 手里的药水[物品](items.md) |
| `ShulkerEntityHelper` | `isClosed()` | `boolean` | 壳是否关闭 |
| | `getAttachedSide()` | `DirectionHelper` | 吸附的方向 |
| | `getColor()` | `DyeColorHelper` 或 `null` | 颜色（未染色为 `null`） |
| `SlimeEntityHelper` | `getSize()` | `number` | 体型大小 |
| | `isSmall()` | `boolean` | 是否小史莱姆（小于 1 不主动攻击） |

单方法速查（都是布尔/数字判断）：

| 类 | 方法 | 说明 |
| --- | --- | --- |
| `BlazeEntityHelper` | `isOnFire()` | 烈焰人只有着火时才能射火球 |
| `GhastEntityHelper` | `isShooting()` | 是否准备喷火球 |
| `PhantomEntityHelper` | `getSize()` | 幻翼体型 |
| `SpiderEntityHelper` | `isClimbing()` | 是否在爬墙 |
| `VexEntityHelper` | `isCharging()` | 是否冲锋中 |

## 其他实体（other）

| 类 | 方法 | 返回 | 说明 |
| --- | --- | --- | --- |
| `AreaEffectCloudEntityHelper` | `getRadius()` | `number` | 效果云半径 |
| | `getColor()` | `number` | 云的颜色 |
| | `getParticleType()` | `string` | 粒子 ID |
| | `isWaiting()` | `boolean` | 是否处于等待阶段 |
| `FallingBlockEntityHelper` | `getOriginBlockPos()` | `BlockPosHelper` | 开始下落的[位置](position.md) |
| | `getBlockState()` | `BlockStateHelper` | 下落的[方块](blocks.md)状态 |
| `InteractionEntityHelper` | `setCanHit(value)` | `void` | 设置本地是否可命中 |
| | `getLastAttacker()` | `EntityHelper` 或 `null` | 最后攻击它的实体 |
| | `getLastInteracted()` | `EntityHelper` 或 `null` | 最后与它交互的实体 |
| | `getWidth()` / `getHeight()` | `number` | 交互区尺寸 |
| | `shouldRespond()` | `boolean` | 是否响应交互 |
| `TntEntityHelper` | `getRemainingTime()` | `number` | 距爆炸的剩余时间（tick） |

## 友好生物（passive）

### 动物基类 AnimalEntityHelper / TameableEntityHelper

`AnimalEntityHelper` 是所有动物的基类：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isFood(item)` | `boolean` | 该物品能否喂食/催情（接受 `ItemHelper` 或 `ItemStackHelper`，两种重载） |
| `canBreedWith(other)` | `boolean` | 能否与另一只动物繁殖 |

`TameableEntityHelper`（狼、猫、鹦鹉的基类）：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isTamed()` | `boolean` | 是否已驯服 |
| `isSitting()` | `boolean` | 是否坐下 |
| `getOwner()` | `string` 或 `null` | 主人 UUID，未驯服为 `null` |
| `isOwner(living)` | `boolean` | 某生物是否它的主人 |

### 马类 AbstractHorseEntityHelper

马、驴、骡、羊驼、骷髅马等的基类：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getOwner()` | `string` 或 `null` | 主人 UUID |
| `isTame()` | `boolean` | 是否驯服 |
| `isSaddled()` | `boolean` | 是否装了鞍 |
| `isAngry()` | `boolean` | 是否发怒 |
| `isBred()` | `boolean` | 是否繁殖出生（非自然生成） |
| `isEating()` | `boolean` | 是否在进食 |
| `canWearArmor()` | `boolean` | 能否穿马铠 |
| `canBeSaddled()` | `boolean` | 能否装鞍 |
| `getInventorySize()` | `number` | 物品栏大小 |
| `getJumpStrengthStat()` | `number` | 跳跃力属性 |
| `getHorseJumpHeight()` | `number` | 估算的最大跳跃高度 |
| `getMaxJumpStrengthStat()` / `getMinJumpStrengthStat()` | `number` | 跳跃力属性上/下限 |
| `getSpeedStat()` | `number` | 速度属性 |
| `getHorseSpeed()` | `number` | 速度（格/秒） |
| `getMaxSpeedStat()` / `getMinSpeedStat()` | `number` | 速度属性上/下限 |
| `getHealthStat()` | `number` | 血量属性（等于 `getMaxHealth()`） |
| `getMaxHealthStat()` / `getMinHealthStat()` | `number` | 血量属性上/下限 |

马类子类的专属方法：

| 类 | 方法 | 返回 | 说明 |
| --- | --- | --- | --- |
| `HorseEntityHelper` | `getVariant()` | `number` | 马的花色变种 |
| `DonkeyEntityHelper` | `hasChest()` | `boolean` | 是否驮着箱子 |
| `LlamaEntityHelper` | `getVariant()` | `string` | 羊驼花色 |
| | `getStrength()` | `number` | 力量（决定箱子容量） |
| | `isTraderLlama()` | `boolean` | 是否流浪商人的羊驼 |

### 狼 WolfEntityHelper

继承 `TameableEntityHelper`（所以也有 `isTamed()`、`getOwner()` 等）。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isBegging()` | `boolean` | 是否在乞食（已驯服且玩家手拿骨头/肉） |
| `getCollarColor()` | `DyeColorHelper` | 项圈颜色 |
| `isAngry()` | `boolean` | 是否发怒 |
| `isWet()` | `boolean` | 是否湿身 |

```javascript
// 检查周围的狼是不是自己的
const myUuid = Player.getPlayer().getUUID()
const wolves = World.getEntities(16, "wolf")
if (wolves) {
    for (const wolf of wolves) {
        if (wolf.isTamed() && wolf.getOwner() === myUuid) {
            Chat.log(`我的狼：坐下=${wolf.isSitting()} 项圈=${wolf.getCollarColor().getName()}`)
        }
    }
}
```

### 猫科与狐狸

| 类 | 方法 | 返回 | 说明 |
| --- | --- | --- | --- |
| `CatEntityHelper` | `isSleeping()` | `boolean` | 是否在睡觉 |
| | `getCollarColor()` | `DyeColorHelper` | 项圈颜色 |
| | `getVariant()` | `string` | 花色变种 |
| `OcelotEntityHelper` | `isTrusting()` | `boolean` | 是否信任玩家（喂过鳕鱼/鲑鱼） |

狐狸 `FoxEntityHelper` 方法较多：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getItemInMouth()` | `ItemStackHelper` | 嘴里叼的[物品](items.md) |
| `isSnowFox()` / `isRedFox()` | `boolean` | 雪狐/红狐 |
| `getOwner()` / `getSecondOwner()` | `string` 或 `null` | 信任的玩家 UUID |
| `canTrust(entity)` | `boolean` | 是否信任某实体 |
| `hasFoundTarget()` | `boolean` | 是否锁定目标准备扑击 |
| `isSitting()` / `isSleeping()` / `isWandering()` | `boolean` | 坐/睡/闲逛 |
| `isDefending()` | `boolean` | 是否在保护另一只狐狸 |
| `isPouncing()` / `isJumping()` | `boolean` | 起跳前蓄力/跳跃中 |
| `isSneaking()` | `boolean` | 是否在潜行接近猎物 |

### 熊猫 PandaEntityHelper

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getMainGene()` / `getHiddenGene()` | `number` | 主/隐性基因 ID |
| `getMainGeneName()` / `getHiddenGeneName()` | `string` | 基因名称 |
| `isMainGeneRecessive()` / `isHiddenGeneRecessive()` | `boolean` | 基因是否隐性 |
| `isIdle()` | `boolean` | 是否发呆 |
| `isSneezing()` | `boolean` | 是否在打喷嚏 |
| `isPlaying()` | `boolean` | 是否在打滚玩耍 |
| `isSitting()` | `boolean` | 是否坐着 |
| `isLyingOnBack()` | `boolean` | 是否仰躺 |
| `isLazy()` / `isWorried()` / `isPlayful()` / `isBrown()` / `isWeak()` / `isAttacking()` | `boolean` | 基因性格判断：懒惰/忧虑/爱玩/棕色/虚弱/凶猛 |

### 鹦鹉与其他动物

| 类 | 方法 | 返回 | 说明 |
| --- | --- | --- | --- |
| `ParrotEntityHelper` | `getVariant()` | `string` | 羽色变种 |
| | `isSitting()` / `isFlying()` / `isStanding()` | `boolean` | 坐/飞/站 |
| | `isPartying()` | `boolean` | 是否随唱片跳舞 |
| | `isSittingOnShoulder()` | `boolean` | 是否停在玩家肩上 |
| `AllayEntityHelper` | `isDancing()` | `boolean` | 是否跳舞 |
| | `canDuplicate()` | `boolean` | 能否复制 |
| | `isHoldingItem()` | `boolean` | 是否拿着物品 |
| `AxolotlEntityHelper` | `getVariantId()` / `getVariantName()` | `number` / `string` | 颜色变种 |
| | `isPlayingDead()` | `boolean` | 是否在装死 |
| | `isFromBucket()` | `boolean` | 是否来自桶 |
| `BeeEntityHelper` | `hasNectar()` | `boolean` | 是否带着花粉 |
| | `isAngry()` | `boolean` | 是否被激怒 |
| | `hasStung()` | `boolean` | 是否已蜇过人 |
| `DolphinEntityHelper` | `hasFish()` | `boolean` | 嘴里是否叼鱼 |
| | `getTreasurePos()` | `BlockPosHelper` | 寻宝目标[位置](position.md)（默认 0 0 0） |
| | `getMoistness()` | `number` | 湿润度 |
| `FrogEntityHelper` | `getVariant()` | `string` | 变种 |
| | `getTarget()` | `EntityHelper` 或 `null` | 舌头攻击目标 |
| | `isCroaking()` | `boolean` | 是否在鸣叫 |
| `GoatEntityHelper` | `isScreaming()` | `boolean` | 是否尖叫山羊 |
| | `hasLeftHorn()` / `hasRightHorn()` / `hasHorns()` | `boolean` | 角是否还在 |
| `MooshroomEntityHelper` | `isShearable()` | `boolean` | 能否剪毛 |
| | `isRed()` / `isBrown()` | `boolean` | 红/棕哞菇 |
| `SheepEntityHelper` | `isSheared()` / `isShearable()` | `boolean` | 已剪毛/能剪毛 |
| | `getColor()` | `DyeColorHelper` | 毛色 |
| | `isJeb()` | `boolean` | 是否 `jeb_` 彩虹羊 |
| `StriderEntityHelper` | `isSaddled()` | `boolean` | 是否装鞍 |
| | `isShivering()` | `boolean` | 是否在寒冷中发抖 |
| `RabbitEntityHelper` | `getVariant()` | `string` | 毛色变种 |
| | `isKillerBunny()` | `boolean` | 是否杀手兔 |

单方法速查：

| 类 | 方法 | 说明 |
| --- | --- | --- |
| `BatEntityHelper` | `isResting()` | 是否倒挂休息 |
| `IronGolemEntityHelper` | `isPlayerCreated()` | 是否玩家搭建 |
| `PigEntityHelper` | `isSaddled()` | 是否装鞍 |
| `PolarBearEntityHelper` | `isAttacking()` | 是否站立攻击 |
| `SnowGolemEntityHelper` | `hasPumpkin()` / `isShearable()` | 是否戴南瓜/能否剪掉 |
| `FishEntityHelper` | `isFromBucket()` | 鱼是否来自桶（鱼类基类） |

### 鱼类变种

| 类 | 方法 | 返回 | 说明 |
| --- | --- | --- | --- |
| `PufferfishEntityHelper` | `getSize()` | `number` | 膨胀状态：0 未膨胀、1 半膨胀、2 完全膨胀 |
| `TropicalFishEntityHelper` | `getVariant()` | `string` | 花纹变种 |
| | `getSize()` | `string` | 变种体型 |
| | `getBaseColor()` / `getPatternColor()` | `number` | 底色/花纹色 |
| | `getVarietyId()` | `number` | 变种 ID |

## 弹射物（projectile）

### 鱼漂 FishingBobberEntityHelper

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `hasCaughtFish()` | `boolean` | 是否有鱼咬钩 |
| `isInOpenWater()` | `boolean` | 是否在开阔水域（能钓到宝藏） |
| `hasEntityHooked()` | `boolean` | 是否钩住了实体 |
| `getHookedEntity()` | `EntityHelper` 或 `null` | 钩住的实体 |

自己的鱼漂用 `Player.getPlayer().getFishingBobber()` 拿最方便（没在钓鱼时为 `null`）。写自动钓鱼脚本时，核心就是循环检查 `hasCaughtFish()`，为真就收竿再抛竿：

```javascript
const bobber = Player.getPlayer().getFishingBobber()
if (bobber && bobber.hasCaughtFish()) {
    Chat.log("咬钩了，收杆！")
}
```

### 其他弹射物

| 类 | 方法 | 返回 | 说明 |
| --- | --- | --- | --- |
| `ArrowEntityHelper` | `getColor()` | `number` | 药箭粒子颜色，无粒子为 `-1` |
| | `isCritical()` | `boolean` | 是否暴击箭 |
| | `getPiercingLevel()` | `number` | 穿透等级（弩的穿透附魔） |
| `TridentEntityHelper` | `hasLoyalty()` | `boolean` | 是否有忠诚附魔 |
| | `isEnchanted()` | `boolean` | 是否附魔 |
| `WitherSkullEntityHelper` | `isCharged()` | `boolean` | 是否蓝色充能头颅 |

## 载具（vehicle）

| 类 | 方法 | 返回 | 说明 |
| --- | --- | --- | --- |
| `BoatEntityHelper` | `isChestBoat()` | `boolean` | 是否运输船 |
| | `getBoatItem()` | `ItemStackHelper` | 对应的船[物品](items.md) |
| | `isInWater()` | `boolean` | 是否浮在水面 |
| | `isOnLand()` | `boolean` | 是否在陆地上 |
| | `isUnderwater()` | `boolean` | 是否沉在水下 |
| | `isInAir()` | `boolean` | 是否在空中 |
| `FurnaceMinecartEntityHelper` | `isPowered()` | `boolean` | 是否有燃料驱动 |
| `TntMinecartEntityHelper` | `isPrimed()` | `boolean` | 是否已点燃 |
| | `getRemainingTime()` | `number` | 距爆炸剩余 tick，未点燃为 `-1` |

---

通用方法（坐标、血量、发光高亮、类型判断）见[实体基础](entities.md)。
