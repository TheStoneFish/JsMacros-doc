---
icon: lucide/backpack
---

# 背包与容器

本章讲解 JsMacros 的背包与容器 API。无论是玩家自己的背包、箱子、熔炉，还是服务器插件做的"菜单界面"，在 JsMacros 里都是一个 `Inventory` 对象（或它的子类）。学会本章后你可以：读懂任意界面的槽位编号、安全地搬运物品、自动操作附魔台/村民交易等专用容器。

物品本身（`ItemStackHelper`）的 NBT、耐久、附魔等细节不在本章展开，请看 [物品与NBT](items.md)。

## 获取背包对象

入口只有一个：`Player.openInventory()`。它返回**当前打开的界面**对应的 `Inventory` 对象——没开任何界面时对应玩家背包，开着箱子时就是箱子。

```javascript
const inv = Player.openInventory()
Chat.log(`类型: ${inv.getType()}`)
```

!!! note "openInventory 不会替你打开界面"
    这个名字有点误导：它只是"获取当前背包/容器的操作句柄"，并不会打开 GUI。想真正弹出背包界面，请再调用 `inv.openGui()`。

### 判断容器种类

不同界面返回不同的子类（如 `PlayerInventory`、`FurnaceInventory`）。三个常用判断手段：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getType()` | `string` | 界面类型名，如 `"Survival Inventory"`、`"3 Row Chest"`、`"Furnace"`；未知界面返回其他字符串或 `"unknown"` |
| `is(...anyOf)` | `boolean` | 类型名是否匹配任意一个，如 `inv.is("Furnace", "Blast Furnace", "Smoker")` |
| `isContainer()` | `boolean` | 是否为容器（而非玩家自己的背包） |
| `getContainerTitle()` | `string` | 容器标题文字（服务器菜单识别的关键） |

```javascript
const inv = Player.openInventory()
if (inv.isContainer()) {
    Chat.log(`容器标题: ${inv.getContainerTitle()}`)
}
if (inv.is("Enchanting Table")) {
    // 这里 inv 就可以安全调用附魔台专属方法
    Chat.log(inv.getRequiredLevels())
}
```

### 类型名对照全表（InvNameToTypeMap）

`getType()` / `is()` 使用的类型名与对应子类如下（逐一对照 d.ts 的 `InvNameToTypeMap`）：

| 类型名 | 对应子类 | 常见界面 |
| --- | --- | --- |
| `1 Row Chest` ~ `9 Row Chest` | `ContainerInventory` | 1~9 行箱子界面（原版箱子 3 行、大箱子 6 行；服务器菜单常见 1~6 行） |
| `3x3 Container` | `ContainerInventory` | 发射器、投掷器 |
| `Hopper` | `ContainerInventory` | 漏斗（1 行 5 格） |
| `Shulker Box` | `ContainerInventory` | 潜影盒 |
| `Anvil` | `AnvilInventory` | 铁砧 |
| `Beacon` | `BeaconInventory` | 信标 |
| `Furnace` / `Blast Furnace` / `Smoker` | `FurnaceInventory` | 熔炉 / 高炉 / 烟熏炉 |
| `Brewing Stand` | `BrewingStandInventory` | 酿造台 |
| `Crafting Table` | `CraftingInventory` | 工作台 |
| `Enchanting Table` | `EnchantInventory` | 附魔台 |
| `Grindstone` | `GrindStoneInventory` | 砂轮 |
| `Loom` | `LoomInventory` | 织布机 |
| `Villager` | `VillagerInventory` | 村民 / 流浪商人交易 |
| `Smithing Table` | `SmithingInventory` | 锻造台 |
| `Cartography Table` | `CartographyInventory` | 制图台 |
| `Stonecutter` | `StoneCutterInventory` | 切石机 |
| `Survival Inventory` | `PlayerInventory` | 玩家生存背包 |
| `Creative Inventory` | `CreativeInventory` | 创造模式背包 |
| `Horse` | `HorseInventory` | 马 / 驴 / 骡 / 骆驼背包 |

## 槽位编号与 getMap()

**不同界面的槽位编号完全不同，永远不要硬编码槽位号。** 例如生存背包里快捷栏是 36~44，而开着箱子时快捷栏会变成另一段编号。正确姿势是先用 `getMap()` 看分区：

```javascript
const inv = Player.openInventory()
Chat.log(`标题: ${inv.getContainerTitle()}  类型: ${inv.getType()}`)
Chat.log(`总槽位: ${inv.getTotalSlots()}  syncId: ${inv.getCurrentSyncId()}`)

const map = inv.getMap()
for (const section of map.keySet()) {
    const slots = Java.from(map.get(section))
    Chat.log(`[${section}] ${slots.length} 格: ${slots.join(", ")}`)
}
```

!!! tip "排查服务器菜单的刚需脚本"
    上面这段就是"菜单界面透视镜"：打开任意服务器菜单后运行一次，把每个分区名和槽位号都打印出来，再决定点哪个格子。配合下面的"打印每格物品"更直观：

    ```javascript
    const inv = Player.openInventory()
    const total = inv.getTotalSlots()
    for (let i = 0; i < total; i++) {
        const stack = inv.getSlot(i)
        if (!stack.isEmpty()) {
            Chat.log(`#${i} [${inv.getLocation(i)}] ${stack.getItemId()} x${stack.getCount()}`)
        }
    }
    ```

### 分区名全表（InvMapType）

`getMap()` 返回 `JavaMap<分区名, 槽位号数组>`。各界面拥有的分区（逐一对照 d.ts 的 `InvMapType` 命名空间）：

| 界面 | 分区名 |
| --- | --- |
| 所有界面共有 | `hotbar`（快捷栏 9 格）、`main`（主背包 27 格） |
| 生存背包 `Survival Inventory` | + `offhand`、`helmet`、`chestplate`、`leggings`、`boots`、`crafting_in`（2x2 合成格）、`craft_out`（合成输出） |
| 创造背包（物品栏标签页） | + `offhand`、四件盔甲、`delete`（销毁物品格） |
| 创造背包（其他标签页） | 仅 `hotbar`、`creative`（展示区） |
| 箱子 / 漏斗 / 潜影盒等 | + `container`（容器区） |
| 熔炉 / 高炉 / 烟熏炉 | + `input`、`output`、`fuel` |
| 酿造台 | + `input`、`output`、`fuel`（烈焰粉） |
| 工作台 | + `input`（3x3）、`output` |
| 附魔台 | + `item`（待附魔物品）、`lapis`（青金石） |
| 织布机 | + `banner`、`dye`、`pattern`、`output` |
| 切石机 | + `input`、`output` |
| 铁砧 / 砂轮 / 锻造台 / 制图台 / 村民交易 | + `input`、`output` |
| 信标 | + `slot`（放矿物的那一格） |
| 马匹 | + `saddle`、`armor`、`container`（驮箱） |

### 分区相关方法

```javascript
const inv = Player.openInventory()
const hotbarSlots = inv.getSlots("hotbar")            // 分区 -> 槽位号数组
const where = inv.getLocation(hotbarSlots[0])         // 槽位号 -> 分区名
const free = inv.findFreeSlot("main", "hotbar")       // 指定分区找空槽
```

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getMap()` | `JavaMap<InvMapId, number[]>` | 分区名 → 槽位号数组 |
| `getSlots(...mapIdentifiers)` | `number[]` | 取一个或多个分区的全部槽位号 |
| `getLocation(slotNum)` | `string \| null` | 槽位属于哪个分区 |
| `getSlotPos(slot)` | `Pos2D` | 槽位在屏幕上的 x/y 坐标 |
| `getSlotUnderMouse()` | `number` | 鼠标当前指向的槽位号 |
| `getTotalSlots()` | `number` | 界面总槽位数 |

## Inventory 基类方法参考

以下方法所有容器通用。除注明外，返回 `Inventory` 的方法都返回自身，可以链式调用。

### 读取槽位与物品

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getSlot(slot)` | `ItemStackHelper` | 指定槽位的物品（空格子返回空物品栈，用 `isEmpty()` 判断） |
| `getHeld()` | `ItemStackHelper` | 鼠标上正拿着的物品 |
| `getItems()` | `JavaList<ItemStackHelper>` | 整个界面的所有物品 |
| `getItems(...mapIdentifiers)` | `JavaList<ItemStackHelper>` | 指定分区的所有物品 |
| `getItemCount()` | `JavaMap<ItemId, number>` | 每种物品 ID → 总数量 |
| `getSelectedHotbarSlotIndex()` | `number` | 当前选中的快捷栏下标（0~8） |
| `setSelectedHotbarSlotIndex(index)` | `void` | 切换选中的快捷栏（0~8） |

```javascript
const inv = Player.openInventory()
const counts = inv.getItemCount()
Chat.log(`钻石数量: ${counts.get("minecraft:diamond") || 0}`)
```

### 查找物品与空槽

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `findItem(item)` | `number[]` | 找出含有该物品的所有槽位；参数可以是物品 ID 字符串或 `ItemStackHelper` |
| `contains(item)` | `boolean` | 界面里是否有该物品；参数同上 |
| `findFreeInventorySlot()` | `number` | 玩家主背包第一个空槽，找不到返回 `-1` |
| `findFreeHotbarSlot()` | `number` | 快捷栏第一个空槽，找不到返回 `-1` |
| `findFreeSlot(...mapIdentifiers)` | `number` | 在指定分区找第一个空槽，找不到返回 `-1` |

```javascript
const inv = Player.openInventory()
const dirtSlots = Java.from(inv.findItem("minecraft:dirt"))
Chat.log(`泥土所在槽位: ${dirtSlots.join(", ")}`)
```

### 点击与移动操作

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `click(slot)` | `Inventory` | 左键点击槽位 |
| `click(slot, mousebutton)` | `Inventory` | 用指定鼠标键点击：`0` 左键 / `1` 右键 / `2` 中键（创造模式克隆） |
| `quick(slot)` | `Inventory` | Shift+左键（快速移动到另一侧） |
| `quickAll(slot)` | `number` | 把与该槽位相同的物品全部 Shift 移过去，返回移动的数量 |
| `quickAll(slot, button)` | `number` | 同上，`button` 为 `0` 或 `1` |
| `grabAll(slot)` | `Inventory` | 双击集齐：手上有不满的一组时，把背包里同类物品吸到手上 |
| `dragClick(slots, mousebutton)` | `Inventory` | 按住拖动分配：`0` 左键把手上物品均分到各槽位，`1` 右键每格放 1 个；`slots` 是槽位号数组 |
| `split(slot1, slot2)` | `Inventory` | 把手上的一组物品对半分到两个槽位（手上必须拿着物品，可能抛异常；不稳定时可用 `dragClick` 替代） |
| `swap(slot1, slot2)` | `Inventory` | 交换两个槽位（**已弃用**：服务器上有时序问题，优先用 `swapHotbar`） |
| `swapHotbar(slot, hotbarSlot)` | `Inventory` | 等价于指着槽位按数字键/F：`hotbarSlot` 取 `0~8`（快捷栏）或 `40`（副手） |
| `dropSlot(slot)` | `Inventory` | 丢出该槽位物品（旧重载；建议用下面带参数的版本明确语义） |
| `dropSlot(slot, stack)` | `Inventory` | 丢出该槽位物品：`stack` 为 `true` 丢整组，`false` 丢单个 |

```javascript
const inv = Player.openInventory()

inv.quick(10)           // shift 点击 10 号槽
inv.click(11)           // 左键点击
inv.click(12, 1)        // 右键点击（拿起一半 / 放下一个）
inv.swapHotbar(13, 0)   // 13 号槽与快捷栏 0 号位交换
inv.dropSlot(14, true)  // 丢出整组
```

### 打开、关闭与其他

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `openGui()` | `void` | 打开这个背包对应的 GUI 界面 |
| `close()` | `void` | 关闭界面 |
| `closeAndDrop()` | `Inventory` | 关闭界面，并把"鼠标上拿着的物品"丢出去 |
| `getRawContainer()` | `IScreen` | 底层界面对象（进阶用，见 [界面与GUI](screen.md)） |
| `getCurrentSyncId()` | `number` | 当前界面的同步 ID；服务器换菜单页时会变，可用来确认没点到旧界面 |

!!! warning "操作前先确认界面还在"
    连续操作容器时，服务器可能随时关闭/切换界面。批量点击前先核对 `getContainerTitle()` 或 `getCurrentSyncId()`，循环中穿插 `Client.waitTick(1)`，避免把点击发到已经关闭的旧界面上。

## 点击模式详解

Minecraft 协议里每次容器点击都有一个 `mode`（见 [wiki.vg 的 Click Window](https://wiki.vg/Protocol#Click_Container)）。`Inventory` 的各个方法就是这些模式的封装，`ClickSlot` 事件里的 `mode` 字段也用同一套编号：

| mode | 协议名 | 游戏内操作 | 对应方法 |
| --- | --- | --- | --- |
| 0 | PICKUP | 左键 / 右键点击 | `click(slot)` / `click(slot, 0或1)` |
| 1 | QUICK_MOVE | Shift + 点击 | `quick(slot)` / `quickAll(slot)` |
| 2 | SWAP | 数字键 1~9 / F 键 | `swapHotbar(slot, 0~8或40)` |
| 3 | CLONE | 中键（仅创造模式） | `click(slot, 2)` |
| 4 | THROW | Q（丢 1 个）/ Ctrl+Q（丢整组） | `dropSlot(slot, false/true)` |
| 5 | QUICK_CRAFT | 按住左/右键拖动 | `dragClick(slots, 0或1)` |
| 6 | PICKUP_ALL | 双击 | `grabAll(slot)` |

新手最常用的三招：

- **搬运一整组**：`quick(slot)`，物品自动去另一侧。
- **拿一半 / 放一个**：`click(slot, 1)` 右键。
- **精确放到某格**：先 `click(来源槽)` 拿起，再 `click(目标槽)` 放下。

## 容器子类速查

以下每个子类都继承了上面的全部基类方法，这里只列各自的**专属方法**。

### PlayerInventory 玩家背包

`getType()` 为 `Survival Inventory`。继承自 `RecipeInventory`（所以也有配方书方法，见下文）。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isInHotbar(slot)` | `boolean` | 槽位是否在快捷栏或副手 |
| `getOffhand()` | `ItemStackHelper` | 副手物品 |
| `getHelmet()` / `getChestplate()` / `getLeggings()` / `getBoots()` | `ItemStackHelper` | 四件装备 |
| `getOutput()` | `ItemStackHelper` | 2x2 合成输出格 |
| `getInput(x, y)` | `ItemStackHelper` | 合成格中 (x, y) 位置的物品，x/y 取 0~1 |
| `getInput()` | `ItemStackHelper[][]` | 整个合成格（二维数组） |
| `getCraftingWidth()` / `getCraftingHeight()` | `number` | 合成区尺寸（2x2） |
| `getCraftingSlotCount()` | `number` | 合成格数量 |

```javascript
const inv = Player.openInventory()
if (inv.is("Survival Inventory")) {
    Chat.log(`副手: ${inv.getOffhand().getItemId()}`)
    Chat.log(`头盔: ${inv.getHelmet().getItemId()}`)
}
```

### ContainerInventory 通用容器

箱子、大箱子、末影箱、潜影盒、漏斗、发射器等都用它。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `findFreeContainerSlot()` | `number` | 容器区第一个空槽，找不到返回 `-1` |

### CreativeInventory 创造背包

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `search(search)` | `this` | 在搜索页搜索物品 |
| `selectSearch()` / `selectInventory()` / `selectHotbar()` | `this` | 切换到搜索 / 物品栏 / 保存的快捷栏标签页 |
| `selectTab(tabName)` | `this` | 按名称切换标签页 |
| `getTabNames()` | `JavaList<string>` | 所有标签页名称 |
| `getTabTexts()` | `JavaList<TextHelper>` | 所有标签页标题文本 |
| `getShownItems()` | `JavaList<ItemStackHelper>` | 当前页展示的物品 |
| `scroll(amount)` / `scrollTo(position)` | `this` | 滚动列表（取值 -1~1 / 0~1） |
| `setCursorStack(stack)` | `this` | 直接把某物品放到鼠标上（创造特权） |
| `setStack(slot, stack)` | `this` | 直接向槽位写入物品（创造特权） |
| `destroyHeldItem()` / `destroyAllItems()` | `this` | 销毁手上物品 / 清空整个背包 |
| `saveHotbar(index)` / `restoreHotbar(index)` | `this` | 保存 / 恢复快捷栏（0~8） |
| `getSavedHotbar(index)` | `JavaList<ItemStackHelper>` | 查看已保存的快捷栏 |
| `isInHotbar(slot)`、`getOffhand()`、四件装备 getter | 同 PlayerInventory | 装备读取 |

### RecipeInventory 配方书容器（基类）

工作台、熔炉系、玩家背包共同的基类，提供配方书能力。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getOutput()` | `ItemStackHelper` | 输出格物品 |
| `getInput(x, z)` | `ItemStackHelper` | 输入格 (x, z) 的物品 |
| `getInput()` | `ItemStackHelper[][]` | 全部输入格（二维数组） |
| `getInputSize()` | `number` | 配方最大输入格数 |
| `getCraftingWidth()` / `getCraftingHeight()` / `getCraftingSlotCount()` | `number` | 合成区尺寸信息 |
| `getCategory()` | `string` | 配方书分类 |
| `getCraftableRecipes()` | `JavaList<RecipeHelper>` | 当前材料能合成的配方 |
| `getRecipes(craftable)` | `JavaList<RecipeHelper> \| null` | 配方列表；`true` 只列可合成的 |
| `isRecipeBookOpened()` | `boolean` | 配方书是否展开 |
| `toggleRecipeBook()` / `setRecipeBook(open)` | `void` | 开关配方书 |

`RecipeHelper`（单条配方）常用方法：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getOutput()` | `ItemStackHelper` | 配方产物 |
| `getIngredients()` | `JavaList<JavaList<ItemStackHelper>>` | 每个格子的可选材料 |
| `craft(craftAll)` | `RecipeHelper` | 合成一次 / 按住 Shift 合满 |
| `canCraft()` / `canCraft(amount)` | `boolean` | 当前材料是否够合成 |
| `getCraftableAmount()` | `number` | 最多能合成几次 |
| `getGroup()` | `string` | 配方分组 |

```javascript
// 站在工作台前打开它再运行：自动合成一组木棍
const inv = Player.openInventory()
if (inv.is("Crafting Table")) {
    const recipes = inv.getCraftableRecipes()
    for (let i = 0; i < recipes.size(); i++) {
        const r = recipes.get(i)
        if (r.getOutput().getItemId() === "minecraft:stick") {
            r.craft(true)
            break
        }
    }
}
```

### CraftingInventory 工作台

继承 `RecipeInventory`，合成区为 3x3（`getInput(x, y)` 的 x/y 取 0~2），无额外专属方法。

### FurnaceInventory 熔炉 / 高炉 / 烟熏炉

继承 `RecipeInventory`。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getSmeltedItem()` | `ItemStackHelper` | 正在烧炼的物品 |
| `getFuel()` | `ItemStackHelper` | 燃料格物品 |
| `canUseAsFuel(stack)` | `boolean` | 某物品能否当燃料 |
| `isSmeltable(stack)` | `boolean` | 某物品能否被烧炼 |
| `getFuelValues()` | `JavaMap<string, number>` | 所有燃料 → 燃烧时长（tick） |
| `getSmeltingProgress()` | `number` | 当前烧炼进度（tick） |
| `getTotalSmeltingTime()` | `number` | 烧炼一个所需总时长（tick） |
| `getRemainingSmeltingTime()` | `number` | 当前这个还差多少 tick 烧完 |
| `getRemainingFuelTime()` / `getTotalFuelTime()` | `number` | 当前燃料剩余 / 总燃烧时长（tick） |
| `isBurning()` | `boolean` | 炉子是否在燃烧 |

### EnchantInventory 附魔台

三个附魔选项的下标为 0、1、2（对应 1/2/3 级栏位）。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getRequiredLevels()` | `number[]` | 三个选项各需要的经验等级 |
| `getEnchantments()` | `TextHelper[]` | 三个选项显示的附魔文本 |
| `getEnchantmentHelpers()` | `EnchantmentHelper[]` | 三个选项的附魔详情（等级上限等） |
| `getEnchantmentIds()` | `string[]` | 三个选项的附魔 ID |
| `getEnchantmentLevels()` | `number[]` | 三个选项的附魔等级 |
| `doEnchant(index)` | `boolean` | 点击第 `index` 个选项附魔，返回是否成功 |
| `getItemToEnchant()` | `ItemStackHelper` | 待附魔的物品 |
| `getLapis()` | `ItemStackHelper` | 青金石格的物品 |

### AnvilInventory 铁砧

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getName()` | `string` | 当前输入的新名字 |
| `setName(name)` | `this` | 设置改名文本（取出成品时生效） |
| `getLevelCost()` | `number` | 本次操作需要的经验等级 |
| `getMaximumLevelCost()` | `number` | 默认等级上限（原版 40 级） |
| `getItemRepairCost()` | `number` | 完全修复所需材料数量 |
| `getLeftInput()` / `getRightInput()` | `ItemStackHelper` | 左 / 右输入格 |
| `getOutput()` | `ItemStackHelper` | 预期产物 |

```javascript
// 铁砧改名：放入物品后运行，再取出成品
const inv = Player.openInventory()
if (inv.is("Anvil")) {
    inv.setName("我的神剑")
    Chat.log(`花费等级: ${inv.getLevelCost()}`)
    inv.quick(inv.getSlots("output")[0])  // shift 取出成品
}
```

### GrindStoneInventory 砂轮

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getTopInput()` / `getBottomInput()` | `ItemStackHelper` | 上 / 下输入格 |
| `getOutput()` | `ItemStackHelper` | 预期产物 |
| `simulateXp()` | `number` | 祛魔预计返还的最少经验（最多约为该值 2 倍） |

### SmithingInventory 锻造台

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getLeftInput()` / `getRightInput()` | `ItemStackHelper` | 左（模板/装备）/ 右（材料）输入格 |
| `getOutput()` | `ItemStackHelper` | 预期产物 |

### StoneCutterInventory 切石机

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getRecipes()` | `JavaList<ItemStackHelper>` | 所有可选配方的产物列表 |
| `getAvailableRecipeCount()` | `number` | 可选配方数量 |
| `selectRecipe(idx)` | `this` | 选中第 `idx` 个配方 |
| `getSelectedRecipeIndex()` | `number` | 当前选中的配方下标 |
| `getOutput()` | `ItemStackHelper` | 选中配方的产物 |
| `canCraft()` | `boolean` | 是否已选中可合成的配方 |

```javascript
// 切石机：选中"石砖"配方并取出
const inv = Player.openInventory()
if (inv.is("Stonecutter")) {
    const recipes = inv.getRecipes()
    for (let i = 0; i < recipes.size(); i++) {
        if (recipes.get(i).getItemId() === "minecraft:stone_bricks") {
            inv.selectRecipe(i)
            Client.waitTick(2)
            inv.quick(inv.getSlots("output")[0])
            break
        }
    }
}
```

### LoomInventory 织布机

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `listAvailablePatterns()` | `JavaList<string>` | 可用图案 ID 列表 |
| `selectPatternId(id)` | `boolean` | 按图案 ID 选择，返回是否成功 |
| `selectPattern(index)` | `boolean` | 按列表下标选择 |
| `selectPatternName(name)` | `boolean` | 按名称选择（**已弃用**，用 `selectPatternId`） |

### CartographyInventory 制图台

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getMapItem()` | `ItemStackHelper` | 地图格物品 |
| `getMaterial()` | `ItemStackHelper` | 材料格（纸等）物品 |
| `getOutput()` | `ItemStackHelper` | 预期产物 |

### BrewingStandInventory 酿造台

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getFirstPotion()` / `getSecondPotion()` / `getThirdPotion()` | `ItemStackHelper` | 三个药水格 |
| `getPotions()` | `JavaList<ItemStackHelper>` | 三个药水格列表 |
| `getIngredient()` | `ItemStackHelper` | 原料格 |
| `getFuel()` | `ItemStackHelper` | 燃料格（烈焰粉） |
| `getFuelCount()` | `number` | 剩余燃料点数 |
| `getMaxFuelUses()` | `number` | 燃料点数上限（固定 20） |
| `getBrewTime()` | `number` | 已酿造时间（tick） |
| `getRemainingTicks()` | `number` | 剩余酿造时间（tick） |
| `canBrewCurrentInput()` | `boolean` | 当前药水+原料能否酿造 |
| `isBrewablePotion(potion)` | `boolean` | 某药水是否可被酿造 |
| `isValidIngredient(ingredient)` | `boolean` | 某物品是否是有效原料 |
| `isValidRecipe(potion, ingredient)` | `boolean` | 药水+原料组合是否有效 |
| `previewPotion(potion, ingredient)` | `ItemStackHelper` | 预览酿造产物 |
| `previewPotions()` | `JavaList<ItemStackHelper>` | 预览当前输入的全部产物 |

### BeaconInventory 信标

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getLevel()` | `number` | 信标层数（1~4） |
| `getFirstEffect()` / `getSecondEffect()` | `string \| null` | 当前主 / 副效果 ID |
| `selectFirstEffect(id)` / `selectSecondEffect(id)` | `boolean` | 选择主 / 副效果（效果 ID，如 `minecraft:speed`） |
| `applyEffects()` | `boolean` | 点击确认按钮应用效果 |

### HorseInventory 马匹

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isSaddled()` / `canBeSaddled()` | `boolean` | 是否已装鞍 / 能否装鞍 |
| `getSaddle()` | `ItemStackHelper` | 鞍格物品 |
| `hasArmorSlot()` | `boolean` | 是否有马铠格 |
| `getArmor()` | `ItemStackHelper` | 马铠格物品 |
| `hasChest()` | `boolean` | 是否装了箱子（驴/骡） |
| `getInventorySize()` | `number` | 驮箱容量 |
| `getHorseInventory()` | `JavaList<ItemStackHelper>` | 驮箱内物品列表 |
| `getHorse()` | `AbstractHorseEntityHelper` | 这个背包属于哪匹马（实体详情见 [实体](entities.md)） |

### VillagerInventory 村民交易

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getTrades()` | `JavaList<TradeOfferHelper>` | 全部交易项 |
| `selectTrade(index)` | `this` | 选中第 `index` 项交易（材料会自动填入输入格） |
| `getExperience()` | `number` | 村民当前经验 |
| `getLevelProgress()` | `number` | 升级进度 |
| `getMerchantRewardedExperience()` | `number` | 交易给玩家的经验 |
| `canRefreshTrades()` | `boolean` | 能否刷新交易 |
| `isLeveled()` | `boolean` | 村民是否有等级（流浪商人没有） |

`TradeOfferHelper`（单条交易）常用方法：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getInput()` | `JavaList<ItemStackHelper>` | 所需的输入物品 |
| `getLeftInput()` / `getRightInput()` | `ItemStackHelper` | 第一 / 第二个输入（含折扣后的实际数量） |
| `getOutput()` | `ItemStackHelper` | 交易产物 |
| `select()` | `TradeOfferHelper` | 在界面上选中这条交易 |
| `isAvailable()` | `boolean` | 是否还能交易（未锁定） |
| `getUses()` / `getMaxUses()` | `number` | 已交易次数 / 锁定前最大次数 |
| `getIndex()` | `number` | 在交易列表中的下标 |
| `getExperience()` | `number` | 本条交易给的经验 |
| `getCurrentPriceAdjustment()` | `number` | 当前价格调整（负数为打折） |
| `getNBT()` | `NBTCompoundHelper` | 交易的 NBT 数据 |

```javascript
// 列出村民交易并完成第一条可用的"绿宝石换物"交易
const inv = Player.openInventory()
if (inv.is("Villager")) {
    const trades = inv.getTrades()
    for (let i = 0; i < trades.size(); i++) {
        const t = trades.get(i)
        Chat.log(`#${i} ${t.getLeftInput().getItemId()} x${t.getLeftInput().getCount()}`
            + ` -> ${t.getOutput().getItemId()} ${t.isAvailable() ? "" : "(已锁定)"}`)
    }
    const first = trades.get(0)
    if (first.isAvailable()) {
        first.select()                          // 选中交易，材料自动填入
        Client.waitTick(5)
        inv.quick(inv.getSlots("output")[0])    // shift 取走产物
    }
}
```

## 相关事件

背包相关的四个事件（完整事件机制见 [事件系统](events.md)）：

| 事件 | 可取消 | 字段 / 方法 | 触发时机 |
| --- | --- | --- | --- |
| `OpenContainer` | 是 | `inventory`（Inventory）、`screen` | 打开任意容器界面 |
| `ClickSlot` | 是 | `mode`、`button`、`slot`、`getInventory()` | 你点击了某个槽位（`mode` 见上文点击模式表） |
| `SlotUpdate` | 否 | `type`（`"HELD"`/`"INVENTORY"`/`"SCREEN"`）、`slot`、`oldStack`、`newStack`、`getInventory()` | 槽位内容发生变化 |
| `DropSlot` | 是 | `slot`、`all`（是否整组）、`getInventory()` | 从界面丢出物品 |

```javascript
// 保护贵重物品：阻止丢出钻石
JsMacros.on("DropSlot", true, JavaWrapper.methodToJava((e, ctx) => {
    const stack = e.getInventory().getSlot(e.slot)
    if (stack.getItemId() === "minecraft:diamond") {
        e.cancel()
        ctx.releaseLock()
        Chat.log("已阻止丢弃钻石")
    }
}))
```

!!! note "joined 回调要尽快放锁"
    `DropSlot`、`ClickSlot` 这类可取消事件需要用 `JsMacros.on(事件名, true, 回调)` 的 joined 形式监听才能 `cancel()`。取消后要尽快 `ctx.releaseLock()`，否则会卡住游戏线程。

## 实战示例

### 示例一：一键清理垃圾物品

把背包里的圆石、泥土、砂砾整组丢出去。绑定到快捷键使用效果最佳（见 [按键绑定](keybind.md)）。

```javascript
const TRASH = ["minecraft:cobblestone", "minecraft:dirt", "minecraft:gravel"]
const inv = Player.openInventory()

let dropped = 0
for (const id of TRASH) {
    const slots = Java.from(inv.findItem(id))
    for (const slot of slots) {
        inv.dropSlot(slot, true)   // true = 丢整组
        dropped++
        Client.waitTick(2)         // 每次操作间隔 2 tick，防止服务器判定异常
    }
}
Chat.log(`清理完成，丢弃 ${dropped} 组`)
```

变体：如果开着箱子，把 `dropSlot` 换成 `quick(slot)` 就是"把垃圾全部塞进箱子"。

### 示例二：自动点击服务器菜单

服务器插件菜单本质是"若干行的箱子界面 + 特定标题 + 特定格子放按钮物品"。套路：先用调试脚本打印标题和格子，再写匹配逻辑。

```javascript
// 常驻脚本：打开标题含"签到"的菜单时，自动点击绿宝石按钮
JsMacros.on("OpenContainer", JavaWrapper.methodToJava((e, ctx) => {
    const inv = e.inventory
    const title = inv.getContainerTitle()
    Chat.log(`打开容器: "${title}" 类型: ${inv.getType()}`)

    if (title.includes("签到")) {
        Client.waitTick(10)   // 等服务器把按钮物品填进菜单
        const slots = inv.findItem("minecraft:emerald")
        if (slots.length > 0) {
            inv.click(slots[0])
            Chat.log(`已点击签到按钮（槽位 ${slots[0]}）`)
        } else {
            Chat.log("没找到绿宝石按钮，请用调试脚本重新确认")
        }
    }
}))
```

!!! tip "按钮找不到？"
    有些菜单按钮的物品 ID 相同、只有名字不同。这时改成遍历 `getSlots("container")`，用 `getSlot(i).getName().getString()` 匹配名字即可。

### 示例三：附魔台自动附魔

打开附魔台、放好物品和青金石后运行：打印三个选项，并自动点 3 级栏位。

```javascript
const inv = Player.openInventory()
if (!inv.is("Enchanting Table")) {
    Chat.log("请先打开附魔台并放入物品")
} else {
    Client.waitTick(5)   // 等附魔选项从服务器同步过来
    const levels = inv.getRequiredLevels()
    const texts = inv.getEnchantments()
    for (let i = 0; i < levels.length; i++) {
        const name = texts[i] ? texts[i].getString() : "?"
        Chat.log(`选项 ${i + 1}: 需要 ${levels[i]} 级 - ${name}`)
    }

    if (inv.doEnchant(2)) {   // 下标 2 = 第三个（最高级）选项
        Chat.log("附魔成功")
    } else {
        Chat.log("附魔失败：检查经验等级、青金石数量")
    }
}
```

## 反作弊与稳定性提示

!!! warning "别把脚本写成连点器"
    服务器（尤其带反作弊插件的）会检测容器操作频率与时序：

    - 循环批量点击时，每次操作之间加 `Client.waitTick(1)`（或 2~3 tick）；跨界面操作用 `Time.sleep(毫秒)` 拉开更大间隔。
    - `quick()` 连续搬同种物品通常安全，但混合 `click` 拿放操作时务必加延迟。
    - `swap()` 已弃用且在服务器上不可靠，用 `swapHotbar()` 代替。
    - 操作前用 `getCurrentSyncId()` / `getContainerTitle()` 确认界面没被切换，服务器换菜单页时旧槽位号全部失效。
    - 自动化交易、自动点菜单是否违规取决于服务器规则，使用前请自行确认。

## 相关页面

- [物品与NBT](items.md)：`ItemStackHelper` 的名称、NBT、耐久、附魔读取
- [玩家](player.md)：`Player` 其余方法、主手交互
- [事件系统](events.md)：事件监听、joined 回调与取消事件
- [按键绑定](keybind.md)：把清理脚本绑到快捷键
- [界面与GUI](screen.md)：`getRawContainer()` 返回的界面对象
