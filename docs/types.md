---
icon: lucide/braces
---

# 常用类型

JsMacros 运行在 JVM 上，脚本里会混用 JavaScript 值、Java 对象和 Minecraft helper。理解这些类型，比死背方法更重要。

## Java 集合

| 类型 | 类似 JS | 常用方法 |
| --- | --- | --- |
| `JavaList<T>` | `Array<T>` | `size()`、`get(i)`、`add(v)` |
| `JavaMap<K, V>` | `Map<K, V>` | `get(k)`、`put(k, v)`、`keySet()` |
| `JavaSet<T>` | `Set<T>` | `contains(v)`、`size()` |
| `JavaArray<T>` | 数组 | `length`、下标访问 |

```javascript
const entities = World.getEntities(8)
if (entities) {
    for (let i = 0; i < entities.size(); i++) {
        Chat.log(entities.get(i).getName().getString())
    }
}
```

## TextHelper

Minecraft 文本不是普通字符串，很多 API 返回 `TextHelper`。

```javascript
const name = Player.getPlayer().getName()
Chat.log(name.getString())
Chat.log(name.getStringStripFormatting())
Chat.log(name.getJson())
```

创建文本：

```javascript
const text = Chat.createTextBuilder()
    .append("Hello ")
    .append("JsMacros")
    .build()

Chat.log(text)
```

## ItemStackHelper

`ItemStackHelper` 表示一组物品。

| 方法 | 作用 |
| --- | --- |
| `getItemId()` | 物品 ID，如 `minecraft:diamond` |
| `getCount()` | 数量 |
| `getName().getString()` | 显示名 |
| `getNBT()` | NBT |
| `isEmpty()` | 是否为空 |
| `copy()` | 复制一份 |
| `isItemEqual(other)` | 比较物品类型 |
| `isNBTEqual(other)` | 比较 NBT |
| `getDurability()` / `getMaxDurability()` | 耐久 |
| `getEnchantments()` | 附魔列表 |

```javascript
const stack = Player.openInventory().getHeld()
if (!stack.isEmpty()) {
    Chat.log(`${stack.getItemId()} x${stack.getCount()}`)
}
```

## 位置和方向

常见位置类型：

| 类型 | 用途 |
| --- | --- |
| `Pos2D` | 二维坐标或向量 |
| `Pos3D` | 三维小数坐标 |
| `BlockPosHelper` | 方块整数坐标，可 `up()`、`north()`、`distanceTo()` |

方向字符串：

```ts
type Direction = "up" | "down" | "north" | "south" | "east" | "west"
```

例如放置方块：

```javascript
const block = Player.getPlayer().getBlockPos().down()
Player.getPlayer().interactBlock(block.getX(), block.getY(), block.getZ(), "up", false)
```

## 槽位和背包分区

`Inventory.getMap()` 会返回分区到槽位列表的映射。常见分区：

| 分区 | 含义 |
| --- | --- |
| `hotbar` | 快捷栏 |
| `main` | 主背包 |
| `offhand` | 副手 |
| `container` | 容器区域 |
| `input` / `output` | 合成、熔炉、交易等输入输出 |
| `fuel` | 熔炉/酿造燃料 |
| `helmet` / `chestplate` / `leggings` / `boots` | 装备槽 |

常见槽位数字类型：

| 类型 | 范围/含义 |
| --- | --- |
| `HotbarSlot` | `0` 到 `8` |
| `OffhandSlot` | `40` |
| `HotbarSwapSlot` | 快捷栏或副手 |
| `ClickSlotButton` | 点击/交换按钮参数 |

## 屏幕名

常见 `HandledScreenName`：

- `Survival Inventory`
- `Creative Inventory`
- `1 Row Chest` 到 `9 Row Chest`
- `Shulker Box`
- `Furnace`、`Blast Furnace`、`Smoker`
- `Crafting Table`
- `Anvil`
- `Villager`
- `Horse`

```javascript
const inv = Player.openInventory()
Chat.log(inv.getType())
```

## 颜色

HUD 和发光颜色通常使用整数 RGB/ARGB。常见写法：

```javascript
const red = 0xFF0000
const green = 0x00FF00
const white = 0xFFFFFF
```

某些方法还会单独接收 `alpha`，范围一般是 `0` 到 `255`。

