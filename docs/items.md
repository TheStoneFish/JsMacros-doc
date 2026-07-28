---
icon: lucide/gem
---

# 物品与NBT

本章是 `ItemStackHelper`（物品堆）的权威参考，同时覆盖 `ItemHelper`（物品类型）、`CreativeItemStackHelper`（创造模式修改）、`EnchantmentHelper`（附魔）、`FoodComponentHelper`（食物）、`NBTElementHelper` 家族（NBT 数据）和 `RecipeHelper`（配方）。所有方法均逐一对照 JsMacros 2.1.0（MC 1.21.1）的类型定义整理。

无论是"检查主手武器耐久"、"按 Lore 找服务器 RPG 装备"，还是"读一把武器的自定义 NBT 标签"，都能在本页找到做法。

## 先分清两个类：ItemStackHelper 与 ItemHelper

| | `ItemStackHelper` | `ItemHelper` |
| --- | --- | --- |
| 代表什么 | **一叠具体的物品**（背包里那格东西） | **物品类型的定义**（"钻石剑"这个概念） |
| 有数量吗 | 有，`getCount()` | 没有，只有上限 `getMaxCount()` |
| 有耐久/附魔/NBT 吗 | 有，每叠各不相同 | 没有，只有"最大耐久""可附魔性"等类型属性 |
| 从哪来 | 背包槽位、主手、掉落物、事件字段 | `stack.getItem()` 或注册表查询 |

两者可以互相转换：

```javascript
const stack = Player.getPlayer().getMainHand() // ItemStackHelper：主手那一叠
const item = stack.getItem()                   // ItemHelper：它的物品类型
const fresh = item.getDefaultStack()           // 又变回 ItemStackHelper：一件全新的默认物品
Chat.log(`${item.getName()} 一组最多 ${item.getMaxCount()} 个`)
```

!!! tip "一句话记忆"
    问"这**一叠**怎么样"（几个、掉了多少耐久、有什么附魔）找 `ItemStackHelper`；问"这**种**物品怎么样"（是不是食物、能不能烧、最大耐久多少）找 `ItemHelper`。

## 从哪里获得物品对象

| 来源 | 写法 | 说明 |
| --- | --- | --- |
| 背包/容器槽位 | `Player.openInventory().getSlot(i)` | 见[背包与容器](inventory.md)，槽位号先用 `getMap()` 查 |
| 主手/副手 | `Player.getPlayer().getMainHand()` / `getOffHand()` | 见[实体](entities.md)的玩家部分 |
| 盔甲栏 | `Player.getPlayer().getHeadArmor()` 等 | 同上 |
| 掉落物实体 | `itemEntity.getContainedItemStack()` | 见[实体](entities.md#itementityhelper) |
| 事件字段 | 如 `HeldItemChange` 的 `item` / `oldItem`，`ItemPickup`、`ItemDamage` 的 `item` | 见[事件系统](events.md)与[全部事件参考](events_reference.md) |
| 注册表（凭 ID 造一个） | `Client.getRegistryManager().getItemStack("diamond_sword")` | 还有 `getItemStack(id, nbt)`、`getItem(id)`（返回 `ItemHelper`）、`getItemIds()` |

```javascript
// 空手也能"造"出一叠物品来查数据（不会真的进背包）
const reg = Client.getRegistryManager()
const stack = reg.getItemStack("netherite_pickaxe")
Chat.log(`${stack.getName().getString()} 最大耐久 ${stack.getMaxDurability()}`)
```

!!! note "物品 ID 可以省略命名空间"
    接受物品/附魔 ID 的参数都是 `CanOmitNamespace` 类型：写 `"diamond_sword"` 和 `"minecraft:diamond_sword"` 等价。模组物品则必须带命名空间。

## ItemStackHelper 方法参考

下面按用途分组，覆盖 d.ts 中 `ItemStackHelper` 的全部方法。

### 识别与基本信息

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getItemId()` | `ItemId` | 物品 ID，如 `"minecraft:diamond_sword"`，**判断物品种类首选** |
| `getName()` | `TextHelper` | 显示名称（含改名/彩色），转字符串用 `.getString()`，见[聊天与文本](chat.md) |
| `getDefaultName()` | `TextHelper` | 物品的默认名称（不受铁砧改名影响） |
| `getCount()` | `number` | 这一叠的数量 |
| `getMaxCount()` | `number` | 这一叠最多能堆几个 |
| `isEmpty()` | `boolean` | 是否为空（空槽位返回的也是 `ItemStackHelper`，先判空再用！） |
| `getItem()` | `ItemHelper` | 取出物品类型定义 |
| `copy()` | `ItemStackHelper` | 复制一份（修改前先复制，避免影响原物品） |
| `getTags()` | `JavaList<ItemTag>` | 物品所属标签，如 `"minecraft:swords"` |
| `getCreativeTab()` | `JavaList<TextHelper>` | 所在创造模式物品栏分组名 |
| `isTool()` | `boolean` | 是否为工具 |
| `isWearable()` | `boolean` | 是否可穿戴（盔甲等） |
| `isFood()` | `boolean` | 是否为食物（详细数据见下文[食物](#foodcomponenthelper)） |

```javascript
const stack = Player.getPlayer().getMainHand()
if (stack.isEmpty()) {
    Chat.log("主手是空的")
} else {
    Chat.log(`ID: ${stack.getItemId()}`)
    Chat.log(`名称: ${stack.getName().getString()}`)
    Chat.log(`数量: ${stack.getCount()} / ${stack.getMaxCount()}`)
}
```

### 判等：equals / isItemEqual 系列的区别

四组方法都有两个重载：既能传 `ItemStackHelper`，也能传原始的 Minecraft `ItemStack` 对象（比如从原生 API 拿到的），用法相同。

| 方法 | 比较范围 | 典型用途 |
| --- | --- | --- |
| `equals(ish)` / `equals(is)` / `equals(obj)` | **最严格**：整叠完全相同（物品与其携带数据都一致） | 判断"就是同一件东西" |
| `isItemEqual(ish)` / `isItemEqual(is)` | 物品种类相同且损伤值相同，**不看数量** | 判断"同种且同耐久" |
| `isItemEqualIgnoreDamage(ish)` / `(is)` | 只看物品种类，**忽略损伤值** | 判断"同种物品"（两把耐久不同的铁镐也算相同） |
| `isNBTEqual(ish)` / `(is)` | 只比较 NBT 数据是否一致 | 判断"数据相同"（如两件同款 RPG 装备） |

!!! tip "只想比种类？直接比 ID 最不容易踩坑"
    上面几个方法的细节语义随游戏版本略有变化（以实际游戏内行为为准）。如果你只关心"是不是钻石剑"，最直白稳妥的写法是：

    ```javascript
    if (stack.getItemId() === "minecraft:diamond_sword") {
        Chat.log("主手是钻石剑")
    }
    ```

```javascript
const a = Player.getPlayer().getMainHand()
const b = Player.getPlayer().getOffHand()
Chat.log(`完全相同: ${a.equals(b)}`)
Chat.log(`同种同耐久: ${a.isItemEqual(b)}`)
Chat.log(`同种(忽略耐久): ${a.isItemEqualIgnoreDamage(b)}`)
Chat.log(`NBT 相同: ${a.isNBTEqual(b)}`)
```

### 耐久与损伤

耐久有两套互补的说法：**durability（还剩多少）** 和 **damage（已经损耗多少）**，满足 `getDurability() + getDamage() = getMaxDurability()`。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getDurability()` | `number` | 当前剩余耐久 |
| `getMaxDurability()` | `number` | 最大耐久 |
| `getDamage()` | `number` | 已损耗的耐久（损伤值） |
| `getMaxDamage()` | `number` | 可承受的最大损伤（等于最大耐久） |
| `getRepairCost()` | `number` | 铁砧修复的累计惩罚成本 |
| `isDamageable()` | `boolean` | 是否有耐久条（土豆没有，别对它调耐久方法） |
| `isUnbreakable()` | `boolean` | 是否带"无法破坏"标记 |
| `setDamage(damage)` | `this` | **已弃用**：请改用 `getCreative().setDamage()`，且建议先 `copy()` |

一个实用的"低耐久报警"脚本（放进服务或长驻脚本，回调用 `JavaWrapper.methodToJava` 包装）：

```javascript
const listener = JsMacros.on("Tick", JavaWrapper.methodToJava(() => {
    const stack = Player.getPlayer().getMainHand()
    if (!stack.isEmpty() && stack.isDamageable() && !stack.isUnbreakable()) {
        const left = stack.getDurability()
        if (left > 0 && left <= 20) {
            Chat.actionbar(`§c注意！${stack.getName().getString()} 耐久仅剩 ${left}`)
        }
    }
}))
// 脚本停止时记得 JsMacros.off(listener)，详见事件系统页
```

### 工具与冷却

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getAttackDamage()` | `number` | 默认攻击伤害 |
| `isSuitableFor(block)` | `boolean` | 用这件工具挖指定方块能否正常掉落（参数可为 `BlockHelper` 或 `BlockStateHelper`，见[方块](blocks.md)） |
| `isOnCooldown()` | `boolean` | 是否在使用冷却中（如末影珍珠、护盾） |
| `getCooldownProgress()` | `number` | 冷却进度 |
| `isEnchantable()` | `boolean` | 是否可附魔 |

### 附魔

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isEnchanted()` | `boolean` | 是否已附魔 |
| `getEnchantments()` | `JavaList<EnchantmentHelper>` | 全部附魔列表 |
| `getEnchantment(id)` | `EnchantmentHelper \| null` | 按 ID 取附魔（含等级），没有则 `null` |
| `hasEnchantment(id)` | `boolean` | 是否带指定 ID 的附魔（**不看等级**） |
| `hasEnchantment(enchantment)` | `boolean` | 是否带指定附魔**且等级相同**（传 `EnchantmentHelper`） |
| `canBeApplied(enchantment)` | `boolean` | 指定附魔能否附到这件物品上 |
| `getPossibleEnchantments()` | `JavaList<EnchantmentHelper>` | 所有可附到此物品的附魔 |
| `getPossibleEnchantmentsFromTable()` | `JavaList<EnchantmentHelper>` | 附魔台能给出的附魔 |

!!! warning "hasEnchantment 两个重载行为不同"
    传**字符串 ID**（如 `"sharpness"`）只检查"有没有这个附魔"；传 **`EnchantmentHelper` 对象**则要求**等级也一致**。查"有没有锋利"用 ID 重载，查"是不是锋利 V"用 `getEnchantment("sharpness")` 再看 `getLevel()` 更直观。

```javascript
const stack = Player.getPlayer().getMainHand()
if (stack.isEnchanted()) {
    for (const e of Java.from(stack.getEnchantments())) {
        Chat.log(`${e.getName()} ${e.getLevel()}/${e.getMaxLevel()} (${e.getId()})`)
    }
    const sharp = stack.getEnchantment("sharpness")
    if (sharp !== null) {
        Chat.log(`锋利等级: ${sharp.getLevel()}`)
    }
} else {
    Chat.log("主手物品没有附魔")
}
```

#### EnchantmentHelper 完整方法表

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getName()` | `string` | 附魔名称（已翻译），如"锋利" |
| `getId()` | `EnchantmentId` | 附魔 ID，如 `"minecraft:sharpness"` |
| `getLevel()` | `number` | 当前等级（从物品上取得时有意义） |
| `getMinLevel()` / `getMaxLevel()` | `number` | 原版可获得的最低/最高等级 |
| `getLevelName(level)` | `string` | 指定等级的完整名称 |
| `getRomanLevelName()` / `getRomanLevelName(level)` | `TextHelper` | 罗马数字等级名（超出 1~3999 时回退为阿拉伯数字） |
| `getWeight()` | `number` | 权重（越高越常见，对应稀有度） |
| `isCursed()` | `boolean` | 是否为诅咒（绑定/消失诅咒） |
| `isTreasure()` | `boolean` | 是否为宝藏附魔（附魔台刷不出，如经验修补） |
| `canBeApplied(item)` | `boolean` | 能否附到指定物品（参数可为 `ItemHelper` 或 `ItemStackHelper`） |
| `getAcceptableItems()` | `JavaList<ItemHelper>` | 所有可接受此附魔的物品 |
| `isCompatible(id)` / `isCompatible(enchantment)` | `boolean` | 与另一附魔是否兼容（可共存） |
| `conflictsWith(id)` / `conflictsWith(enchantment)` | `boolean` | 与另一附魔是否冲突（如锋利 vs 亡灵杀手） |
| `getConflictingEnchantments()` / `(ignoreType)` | `JavaList<EnchantmentHelper>` | 所有冲突附魔（默认只查同目标类型，传 `true` 查全部） |
| `getCompatibleEnchantments()` / `(ignoreType)` | `JavaList<EnchantmentHelper>` | 所有可共存附魔（同上） |

```javascript
// 查一个附魔的"档案"
const stack = Player.getPlayer().getMainHand()
const e = stack.getEnchantment("mending")
if (e !== null) {
    Chat.log(`${e.getName()} 宝藏附魔: ${e.isTreasure()} 诅咒: ${e.isCursed()}`)
    Chat.log(`与经验修补冲突的附魔:`)
    for (const c of Java.from(e.getConflictingEnchantments())) {
        Chat.log(`- ${c.getName()}`)
    }
}
```

### 食物与 FoodComponentHelper

`ItemStackHelper` 上只有 `isFood()` 判断；营养数据挂在物品类型上，通过 `getItem().getFood()` 获取：

| FoodComponentHelper 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getHunger()` | `number` | 恢复的饥饿值（鸡腿图标半格为 1） |
| `getSaturation()` | `number` | 恢复的饱和度 |
| `isAlwaysEdible()` | `boolean` | 不饿时能否食用（如金苹果） |

```javascript
const stack = Player.getPlayer().getMainHand()
if (stack.isFood()) {
    const food = stack.getItem().getFood()
    Chat.log(`饥饿 +${food.getHunger()}  饱和 +${food.getSaturation()}  随时可吃: ${food.isAlwaysEdible()}`)
} else {
    Chat.log("这不是食物")
}
```

### NBT 与数据组件

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getNBT()` | `NBTElementHelper$NBTCompoundHelper` | 物品携带的 NBT 数据，用法见下文 [NBT 详解](#nbt-详解) |

!!! note "1.21 的物品数据已经组件化"
    从 MC 1.20.5 起，原版把物品上的自由 NBT 改成了结构化的**数据组件**（data components），自定义数据一般收进 `minecraft:custom_data` 组件里。JsMacros 2.1.0 的 d.ts 中物品数据入口仍然只有 `getNBT()` 一个，没有单独的组件 API——你拿到的键名结构可能与老版本教程不同（例如出现 `minecraft:custom_data` 这样的键）。**以你实际打印出来的结构为准**，别照抄 1.20 之前的路径。

### Lore 与显示

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getLore()` | `JavaList<TextHelper>` | 全部 Lore 行（物品说明文字）。返回的是**副本**，改它不影响原物品；要真正修改用 `getCreative().addLore()` |
| `areEnchantmentsHidden()` | `boolean` | 附魔是否被隐藏（HideFlags） |
| `areModifiersHidden()` | `boolean` | 属性修饰符是否被隐藏 |
| `isUnbreakableHidden()` | `boolean` | "无法破坏"是否被隐藏 |
| `isCanDestroyHidden()` | `boolean` | "可破坏"标记是否被隐藏 |
| `isCanPlaceHidden()` | `boolean` | "可放置于"标记是否被隐藏 |
| `isDyeHidden()` | `boolean` | 皮革染色是否被隐藏 |

`getLore()` 返回 `TextHelper` 列表，逐行 `.getString()` 得到纯文本（服务器 RPG 装备的属性通常都写在这里），见下文[实战示例](#lore-rpg)。

### 冒险模式限制

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `hasDestroyRestrictions()` | `boolean` | 是否设置了"只能破坏指定方块"（仅冒险模式生效） |
| `hasPlaceRestrictions()` | `boolean` | 是否设置了"只能放置在指定方块上"（仅冒险模式生效） |
| `getDestroyRestrictions()` | `JavaList<BlockPredicateHelper>` | 可破坏方块的过滤器列表 |
| `getPlaceRestrictions()` | `JavaList<BlockPredicateHelper>` | 可放置目标的过滤器列表 |

### 进入创造修改

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getCreative()` | `CreativeItemStackHelper` | 获得可修改版本（详见下文 [CreativeItemStackHelper](#creativeitemstackhelper)） |

### 构造器

一般不需要手动构造（用注册表更方便），但 d.ts 提供了：`new ItemStackHelper(id, count)`（按 ID 与数量）和包装原始 `ItemStack` 的重载。

## ItemHelper 方法全表

`ItemHelper` 描述**物品类型**，从 `stack.getItem()` 或 `Client.getRegistryManager().getItem(id)` 获得。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getName()` | `string` | 物品名称（当前语言，注意这里是字符串不是 `TextHelper`） |
| `getId()` | `ItemId` | 物品 ID |
| `getMaxCount()` | `number` | 一组最多几个 |
| `getMaxDurability()` | `number` | 最大耐久 |
| `isDamageable()` | `boolean` | 这种物品有没有耐久 |
| `isFireproof()` | `boolean` | 是否防火（下界合金系） |
| `isTool()` | `boolean` | 是否为工具 |
| `isWearable()` | `boolean` | 是否可穿戴 |
| `isFood()` | `boolean` | 是否为食物 |
| `getFood()` | `FoodComponentHelper \| null` | 食物数据，非食物返回 `null` |
| `isBlockItem()` | `boolean` | 是否有对应方块（如圆石有、剑没有） |
| `getBlock()` | `BlockHelper \| null` | 对应方块，没有则 `null`，见[方块](blocks.md) |
| `getMiningSpeedMultiplier(state)` | `number` | 对指定方块状态的挖掘速度倍率（默认 `1`） |
| `isSuitableFor(block)` | `boolean` | 挖该方块能否正常掉落（`BlockHelper` / `BlockStateHelper` 两个重载） |
| `canBeRepairedWith(stack)` | `boolean` | 传入的物品能否作为修复材料（如钻石修钻石镐） |
| `getEnchantability()` | `number` | 附魔能力值（越高越容易出好附魔，金质装备就很高），默认 `0` |
| `hasRecipeRemainder()` | `boolean` | 合成后是否留下残留物（如牛奶桶留空桶） |
| `getRecipeRemainder()` | `ItemStackHelper \| null` | 残留物，没有则 `null` |
| `canBeNested()` | `boolean` | 能否放入收纳袋/潜影盒（潜影盒本身就不能套潜影盒） |
| `getCreativeTab()` | `JavaList<TextHelper>` | 创造模式物品栏分组名 |
| `getGroupIcon()` | `JavaList<ItemStackHelper> \| null` | 分组图标物品，无分组时 `null` |
| `getDefaultStack()` | `ItemStackHelper` | 生成一件数量为 1 的默认物品 |
| `getStackWithNbt(nbt)` | `ItemStackHelper` | 生成一件带指定 NBT（SNBT 字符串）的物品，NBT 格式不合法会抛异常 |

```javascript
// 遍历注册表：列出所有防火物品
const reg = Client.getRegistryManager()
for (const item of Java.from(reg.getItems())) {
    if (item.isFireproof()) {
        Chat.log(`${item.getId()} - ${item.getName()}`)
    }
}
```

## CreativeItemStackHelper：创造模式修改物品

`stack.getCreative()` 返回 `CreativeItemStackHelper`，它继承 `ItemStackHelper` 的全部读取方法，并新增一批**修改**方法，全部返回自身、支持链式调用。

!!! warning "只在创造模式有意义"
    这些方法对应创造模式"客户端可直接编辑物品数据"的能力：改完的物品要通过**创造模式背包**的 `setStack()` / `setCursorStack()` 放进游戏才生效。生存模式下服务器不会承认客户端凭空修改的物品数据——不要指望用它在生存服"刷"装备。另外，`getCreative()` 修改的就是原物品堆，想保留原样先 `copy()`：`stack.copy().getCreative()`。

| 方法 | 说明 |
| --- | --- |
| `setDamage(damage)` | 设置损伤值 |
| `setDurability(durability)` | 设置剩余耐久 |
| `setCount(count)` | 设置数量 |
| `setName(name)` | 改名（`string` 或 `TextHelper` 两个重载） |
| `addEnchantment(id, level)` | 按 ID 加附魔（可超原版等级上限） |
| `addEnchantment(enchantment)` | 加一个 `EnchantmentHelper` 附魔 |
| `removeEnchantment(id)` / `removeEnchantment(enchantment)` | 移除附魔 |
| `clearEnchantments()` | 清空全部附魔 |
| `setLore(...lore)` | 用给定内容**替换**全部 Lore |
| `addLore(...lore)` | **追加** Lore 行 |
| `clearLore()` | 清空 Lore |
| `setUnbreakable(unbreakable)` | 设置"无法破坏" |
| `hideEnchantments(hide)` | 隐藏/显示附魔 |
| `hideModifiers(hide)` | 隐藏/显示属性修饰符 |
| `hideUnbreakable(hide)` | 隐藏/显示"无法破坏" |
| `hideCanDestroy(hide)` | 隐藏/显示"可破坏"列表 |
| `hideCanPlace(hide)` | 隐藏/显示"可放置于"列表 |
| `hideDye(hide)` | 隐藏/显示皮革染色 |

完整示例——创造模式下造一把展示用神器并放进快捷栏：

```javascript
// 前提：创造模式，且当前打开的是创造背包（物品栏标签页）
const inv = Player.openInventory()
if (!inv.is("Creative Inventory")) {
    Chat.log("请切到创造模式并打开背包再运行")
} else {
    const stack = Client.getRegistryManager().getItemStack("diamond_sword")
    stack.getCreative()
        .setName("§b§l雷霆之刃")
        .addEnchantment("sharpness", 10)
        .addEnchantment("unbreaking", 3)
        .setLore("§7蕴含风暴之力的试作型武器", "§8—— 由 JsMacros 锻造")
        .setUnbreakable(true)
        .hideUnbreakable(true)

    inv.selectInventory() // 切到物品栏标签页
    const hotbar = Java.from(inv.getMap().get("hotbar"))
    inv.setStack(hotbar[0], stack) // 放到快捷栏第 1 格
    Chat.log("已放入快捷栏")
}
```

## NBT 详解 {#nbt-详解}

服务器玩家的刚需：RPG 服的装备属性、菜单物品的标记、宠物蛋的数据……都藏在物品 NBT 里。入口是 `stack.getNBT()`，返回 `NBTElementHelper$NBTCompoundHelper`（复合标签）。

### NBTElementHelper：类型判断与转换

NBT 是树形结构，每个节点都是一个 `NBTElementHelper`。用法固定三步：**判断类型 → 转成对应子类 → 读值**。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isCompound()` | `boolean` | 是否为复合标签（键值对，最常见） |
| `isList()` | `boolean` | 是否为列表 |
| `isNumber()` | `boolean` | 是否为数字 |
| `isString()` | `boolean` | 是否为字符串 |
| `isNull()` | `boolean` | 是否为空 |
| `asCompoundHelper()` | `NBTCompoundHelper` | 转复合标签（先用 `isCompound()` 确认） |
| `asListHelper()` | `NBTListHelper` | 转列表（先用 `isList()` 确认） |
| `asNumberHelper()` | `NBTNumberHelper` | 转数字（先用 `isNumber()` 确认） |
| `asString()` | `string` | 字符串节点返回其值；其他节点返回 toString 表示（**万能打印**） |
| `asText()` | `TextHelper` | 带格式的美化文本版（适合直接 `Chat.log`） |
| `getType()` | `number` | NBT 类型编号（见下表） |
| `resolve(nbtPath)` | `JavaList<NBTElementHelper> \| null` | 按 [NBT 路径](https://minecraft.wiki/w/NBT_path_format)取节点，找不到返回 `null`，格式错误抛异常 |

静态方法：`NBTElementHelper.wrap(element)`、`wrapCompound(compound)`、`resolve(element)` 用于包装原始 NBT 对象，一般用不到。

`getType()` / `getHeldType()` 返回的类型编号（原版 NBT 标准）：

| 编号 | 类型 | 编号 | 类型 |
| --- | --- | --- | --- |
| 0 | END（空） | 7 | BYTE_ARRAY |
| 1 | BYTE | 8 | STRING |
| 2 | SHORT | 9 | LIST |
| 3 | INT | 10 | COMPOUND |
| 4 | LONG | 11 | INT_ARRAY |
| 5 | FLOAT | 12 | LONG_ARRAY |
| 6 | DOUBLE | | |

### NBTCompoundHelper（复合标签）

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getKeys()` | `JavaSet<string>` | 全部键名 |
| `has(key)` | `boolean` | 是否存在某键 |
| `get(key)` | `NBTElementHelper \| null` | 取子节点 |
| `getType(key)` | `number` | 某键的值类型编号 |
| `asString(key)` | `string` | 直接把某键的值转字符串 |

### NBTListHelper（列表）

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `length()` | `number` | 元素个数 |
| `get(index)` | `NBTElementHelper \| null` | 按下标取元素 |
| `getHeldType()` | `number` | 元素的类型编号 |
| `isPossiblyUUID()` | `boolean` | 是否可能是 UUID（4 个 int 的列表） |
| `asUUID()` | `java.util.UUID \| null` | 转成 UUID |

### NBTNumberHelper（数字）

| 方法 | 说明 |
| --- | --- |
| `asInt()` / `asLong()` / `asShort()` / `asByte()` | 转整数 |
| `asFloat()` / `asDouble()` | 转浮点数 |
| `asNumber()` | 转通用数字 |

### 完整示例：读一把武器的自定义 NBT 并打印

把主手物品的 NBT 整棵树递归打印出来——排查服务器装备数据时，先跑这个看结构：

```javascript
function dumpNBT(el, indent) {
    if (el === null || el.isNull()) return
    if (el.isCompound()) {
        const comp = el.asCompoundHelper()
        for (const key of comp.getKeys()) {
            const child = comp.get(key)
            if (child.isCompound() || child.isList()) {
                Chat.log(`${indent}${key}:`)
                dumpNBT(child, indent + "  ")
            } else {
                Chat.log(`${indent}${key} = ${child.asString()}`)
            }
        }
    } else if (el.isList()) {
        const list = el.asListHelper()
        for (let i = 0; i < list.length(); i++) {
            const child = list.get(i)
            if (child.isCompound() || child.isList()) {
                Chat.log(`${indent}[${i}]:`)
                dumpNBT(child, indent + "  ")
            } else {
                Chat.log(`${indent}[${i}] = ${child.asString()}`)
            }
        }
    } else {
        Chat.log(`${indent}${el.asString()}`)
    }
}

const stack = Player.getPlayer().getMainHand()
const nbt = stack.getNBT()
if (!nbt || nbt.isNull()) {
    Chat.log("这件物品没有 NBT 数据")
} else {
    Chat.log(`=== ${stack.getName().getString()} 的 NBT ===`)
    dumpNBT(nbt, "")
}
```

看清结构后，就可以用 `resolve()` 按路径精准取值（键名含冒号等特殊字符时要加双引号）：

```javascript
const nbt = Player.getPlayer().getMainHand().getNBT()
if (nbt && !nbt.isNull()) {
    // 路径以上一步打印出来的实际结构为准，下面只是示例
    const found = nbt.resolve('"minecraft:custom_data".rpg_id')
    if (found !== null && !found.isEmpty()) {
        Chat.log(`rpg_id = ${found.get(0).asString()}`)
    } else {
        Chat.log("没有这个标签")
    }
}
```

!!! tip "想快速看全貌，还可以直接美化输出"
    `Chat.log(nbt.asText())` 会输出带格式的整棵 NBT 树，适合粗看；精确取值再用 `get()` / `resolve()`。

## 实战：按 Lore 找服务器 RPG 装备

服务器 RPG 装备的属性（品质、词条）几乎都写在 Lore 里。`getLore()` 返回 `TextHelper` 列表，逐行 `getString()` 就能做文字匹配。下面遍历当前背包，找出所有 Lore 含关键字的物品：

```javascript
const keyword = "传说" // 改成你要找的文字，如"神话"、"绑定"
const inv = Player.openInventory()
const total = inv.getTotalSlots()
let hits = 0

for (let i = 0; i < total; i++) {
    const stack = inv.getSlot(i)
    if (stack.isEmpty()) continue

    for (const line of Java.from(stack.getLore())) {
        const text = line.getString()
        if (text.includes(keyword)) {
            hits++
            Chat.log(`#${i} [${inv.getLocation(i)}] ${stack.getName().getString()}`)
            Chat.log(`    ${text}`)
            break // 这件物品已命中，看下一件
        }
    }
}
Chat.log(hits > 0 ? `共找到 ${hits} 件` : `背包里没有 Lore 含"${keyword}"的物品`)
```

!!! note "颜色代码不影响匹配"
    `getString()` 返回的是**剥掉格式后的纯文本**，Lore 里的 `§c` 之类颜色代码不会混进来，直接 `includes()` 匹配即可。槽位编号与 `getLocation()` 的含义见[背包与容器](inventory.md)。

## RecipeHelper：配方

在**玩家背包、工作台、熔炉**这类带配方书的界面里（`RecipeInventory` 子类），可以列出并直接合成配方：`inv.getCraftableRecipes()`（当前可合成的）或 `inv.getRecipes(craftable)`。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getOutput()` | `ItemStackHelper` | 配方产物 |
| `getIngredients()` | `JavaList<JavaList<ItemStackHelper>>` | 原料表（每格是一组可替换物品） |
| `craft(craftAll)` | `RecipeHelper` | 合成：`false` 合一次，`true` 合到不能再合 |
| `canCraft()` / `canCraft(amount)` | `boolean` | 当前背包能否合成（指定数量） |
| `getCraftableAmount()` | `number` | 当前背包最多能合几次 |
| `getGroup()` | `string` | 配方分组 |

```javascript
// 打开背包或工作台后运行：列出现在能合成什么
const inv = Player.openInventory()
if (inv.is("Survival Inventory", "Crafting Table")) {
    for (const r of Java.from(inv.getCraftableRecipes())) {
        const out = r.getOutput()
        Chat.log(`${out.getName().getString()} x${out.getCount()}（最多 ${r.getCraftableAmount()} 次）`)
    }
    // 找到想要的配方后：r.craft(false) 合一次；r.craft(true) 合到底
} else {
    Chat.log("请在玩家背包或工作台界面运行")
}
```

## 相关页面

- [背包与容器](inventory.md)：槽位编号、getMap 分区、点击与搬运物品
- [聊天与文本](chat.md)：`TextHelper` 的完整用法（`getName()`/`getLore()` 都返回它）
- [实体](entities.md)：主手/副手/盔甲、掉落物实体
- [方块](blocks.md)：`isSuitableFor` 用到的 `BlockHelper` / `BlockStateHelper`
- [事件系统](events.md)与[全部事件参考](events_reference.md)：携带物品字段的事件
