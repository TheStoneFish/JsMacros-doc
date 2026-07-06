---
icon: lucide/backpack
---

# 背包

本章先讲 JsMacros 最常用的背包 API。目标是先掌握两件事：看懂槽位，安全搬运物品。

## 获取背包对象

背包入口是 `Player.openInventory()`。

```javascript
const inv = Player.openInventory()
```

返回值是 `Inventory`，在不同界面下可能是它的子类，比如 `PlayerInventory` 或 `ContainerInventory`。

```javascript
const inv = Player.openInventory()
if (inv.isContainer()) {
    Chat.log(`容器标题: ${inv.getContainerTitle()}`)
}
```

## 先看槽位映射

不同 GUI 的槽位编号不同，不要直接硬编码。先看映射。

```javascript
const inv = Player.openInventory()
Chat.log(`type: ${inv.getType()}`)
Chat.log(`${inv.getMap()}`)
```

`getMap()` 会返回分区到槽位数组的映射。常见分区名：

- `hotbar` 快捷栏
- `main` 主背包
- `offhand` 副手
- `container` 容器区
- `input` / `output` 输入输出槽

更多分区：

| 分区 | 常见界面 |
| --- | --- |
| `fuel` | 熔炉、酿造台 |
| `lapis` / `item` | 附魔台 |
| `saddle` / `armor` | 马 |
| `helmet` / `chestplate` / `leggings` / `boots` | 玩家装备 |
| `crafting_in` / `craft_out` | 生存背包合成格 |

## 槽位查询

```javascript
const inv = Player.openInventory()
const hotbarSlots = inv.getSlots("hotbar")
const free = inv.findFreeSlot("main", "hotbar")
const location = inv.getLocation(hotbarSlots[0])
```

| 方法 | 作用 |
| --- | --- |
| `getSlots(...mapIdentifiers)` | 取某些分区的槽位数组 |
| `getLocation(slot)` | 查询槽位属于哪个分区 |
| `findFreeInventorySlot()` | 找玩家背包空槽 |
| `findFreeHotbarSlot()` | 找快捷栏空槽 |
| `findFreeSlot(...mapIdentifiers)` | 在指定分区找空槽 |
| `getTotalSlots()` | 总槽位数 |
| `getSlotUnderMouse()` | 鼠标下槽位 |
| `getSlotPos(slot)` | 槽位屏幕坐标 |

## 读取物品信息

```javascript
const inv = Player.openInventory()
const slot = inv.getSelectedHotbarSlotIndex()
const stack = inv.getSlot(slot)
Chat.log(`slot=${slot}, id=${stack.getItemId()}, count=${stack.getCount()}`)
```

查找和统计也很常用：

```javascript
const inv = Player.openInventory()
const counts = inv.getItemCount()
const dirtSlots = inv.findItem("minecraft:dirt")
Chat.log(`dirt slots: ${dirtSlots}`)
```

`getItemCount()` 返回 `JavaMap<ItemId, number>`：

```javascript
const counts = inv.getItemCount()
Chat.log(counts.get("minecraft:diamond") || 0)
```

## ItemStackHelper 速查

```javascript
const stack = Player.openInventory().getHeld()
if (!stack.isEmpty()) {
    Chat.log(`${stack.getName().getString()} ${stack.getItemId()} x${stack.getCount()}`)
}
```

| 方法 | 作用 |
| --- | --- |
| `getItemId()` | 物品 ID |
| `getName()` / `getDefaultName()` | 显示名 |
| `getCount()` / `getMaxCount()` | 数量 |
| `getNBT()` | NBT |
| `getDurability()` / `getMaxDurability()` | 耐久 |
| `getDamage()` / `setDamage(damage)` | 损伤值 |
| `getEnchantments()` | 附魔 |
| `isEmpty()` | 是否空物品 |
| `isFood()` / `isTool()` / `isWearable()` | 类型判断 |
| `copy()` | 复制物品栈 |

## 常见操作

```javascript
const inv = Player.openInventory()

inv.quick(10)          // shift 点击
inv.click(11)          // 左键点击
inv.click(12, 1)       // 右键点击
inv.swapHotbar(13, 0)  // 与快捷栏 0 号位交换
inv.dropSlot(14, true) // 丢弃整组
```

注意：

- `swap(slot1, slot2)` 在服务器上可能不稳定，优先使用 `swapHotbar`。
- 连续大量操作时，建议在循环里加 `Client.waitTick(1)`，减少不同步。
- 操作容器前先确认 `inv.getContainerTitle()` 或 `inv.getCurrentSyncId()`，避免点到已经关闭的旧界面。

## 示例：把容器中的第一组圆石搬到背包

```javascript
const inv = Player.openInventory()
if (!inv.isContainer()) {
    Chat.log("请先打开容器界面")
    return
}

const slots = inv.findItem("minecraft:cobblestone")
if (slots.length > 0) {
    inv.quick(slots[0])
    Chat.log(`已搬运槽位 ${slots[0]} 的圆石`)
} else {
    Chat.log("容器中没有圆石")
}
```

## PlayerInventory 专用方法

玩家背包界面有装备和 2x2 合成格：

```javascript
const inv = Player.openInventory()
Chat.log(inv.getOffhand().getItemId())
Chat.log(inv.getHelmet().getItemId())
```

| 方法 | 作用 |
| --- | --- |
| `isInHotbar(slot)` | 槽位是否快捷栏 |
| `getOffhand()` | 副手 |
| `getHelmet()` / `getChestplate()` / `getLeggings()` / `getBoots()` | 装备 |
| `getInput()` | 合成输入格 |
| `getOutput()` | 合成输出 |
| `getCraftingWidth()` / `getCraftingHeight()` | 合成区域尺寸 |

## ContainerInventory 专用方法

```javascript
const inv = Player.openInventory()
if (inv.isContainer()) {
    const free = inv.findFreeContainerSlot()
    Chat.log(`容器空槽: ${free}`)
}
```

配方书类界面还可能有：

| 方法 | 作用 |
| --- | --- |
| `search(search)` | 搜索配方/物品 |
| `selectSearch()` | 选中搜索框 |
| `selectInventory()` | 回到背包 |
| `getShownItems()` | 当前显示物品 |
| `scroll(amount)` / `scrollTo(position)` | 滚动 |

## 示例：保护贵重物品不被丢弃

```javascript
JsMacros.on("DropSlot", true, JavaWrapper.methodToJava((e, context) => {
    const inv = Player.openInventory()
    const stack = inv.getSlot(e.slot)
    if (stack.getItemId() === "minecraft:diamond") {
        e.cancel()
        context.releaseLock()
        Chat.log("已阻止丢弃钻石")
    }
}))
```

`DropSlot` 是可取消事件。joined 回调里取消后要尽快 `releaseLock()`。

后续还可以继续补：常见界面的槽位对照图、自动整理模板，以及容器和玩家背包的双向搬运策略。
