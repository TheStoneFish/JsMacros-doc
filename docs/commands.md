---
icon: lucide/terminal
---

# 自定义命令

JsMacros 允许你注册**客户端命令**：像 `/waypoint`、`/calc` 这样在聊天框输入的命令，由你的脚本在本地处理，支持参数解析和 Tab 自动补全，体验和原版命令完全一致。

!!! note "客户端命令意味着什么"
    命令只存在于你自己的客户端：服务器不知道它，其他玩家也没有。输入未注册的命令仍会发给服务器（通常得到 Unknown command）。已注册的命令会被本地拦截执行，不会发包。

## 入口

| 方法 | 说明 |
| --- | --- |
| `Chat.getCommandManager()` | 返回 `CommandManager`，推荐入口 |
| `CommandManager.createCommandBuilder(name)` | 创建命令构建器，`name` 是命令名（不带 `/`） |
| `Chat.createCommandBuilder(name)` | 旧入口，已弃用，请改用上面两行 |
| `Chat.unregisterCommand(name)` / `Chat.reRegisterCommand(node)` | 同样已弃用，改用 `CommandManager` 里的同名方法 |

## 快速上手

```javascript
const manager = Chat.getCommandManager()

manager.createCommandBuilder("hello")
    .executes(JavaWrapper.methodToJava((ctx) => {
        Chat.log("§aHello JsMacros!")
        return true
    }))
    .register()
```

保存运行后，在聊天框输入 `/hello` 即可。完整流程是固定的五步：

1. `createCommandBuilder(name)` 创建构建器；
2. 用 `xxxArg(...)` 链式声明参数（可选）；
3. 用 `suggest*` 提供自动补全（可选）；
4. `executes(callback)` 挂执行回调，回调收到 `CommandContextHelper`；
5. `register()` 注册。

!!! warning "回调要返回 true"
    官方 JSDoc 明确要求：`executes` 的回调最后要 `return true` 表示执行成功。

## 参数类型全表

以下方法全部来自 `CommandBuilder`，每个都返回自身，可以继续链式调用。参数 `name` 是取值时用的参数名。

**基础与字符串：**

| 方法 | 输入示例 | 说明 |
| --- | --- | --- |
| `literalArg(name)` | `add` | 固定字面量，用来做子命令，不产生可读取的参数值 |
| `wordArg(name)` | `home1` | 单个单词（不含空格） |
| `quotedStringArg(name)` | `"带 空格"` | 单词或带引号的字符串 |
| `greedyStringArg(name)` | `剩下的 全部 内容` | 贪婪字符串，吞掉行尾所有内容，必须是最后一个参数 |
| `regexArgType(name, regex, flags)` | — | 按正则匹配，`flags` 如 `"i"` |
| `booleanArg(name)` | `true` / `false` | 布尔值 |
| `uuidArgType(name)` | `069a79f4-...` | UUID |

**数字：**

| 方法 | 说明 |
| --- | --- |
| `intArg(name)` / `intArg(name, min, max)` | 整数，可限定范围 |
| `longArg(name)` / `longArg(name, min, max)` | 长整数，可限定范围 |
| `doubleArg(name)` / `doubleArg(name, min, max)` | 浮点数，可限定范围 |
| `intRangeArg(name)` | 整数范围写法，如 `3..10` |
| `floatRangeArg(name)` | 小数范围写法 |
| `angleArg(name)` | 角度（支持 `~` 相对写法） |
| `timeArg(name)` | 时间，如 `10t`、`5s`、`2d` |

**游戏对象：**

| 方法 | 说明 |
| --- | --- |
| `blockPosArg(name)` | 方块坐标，支持 `~ ~ ~` |
| `columnPosArg(name)` | 柱坐标（x z 两个分量） |
| `blockArg(name)` / `blockStateArg(name)` / `blockPredicateArg(name)` | 方块 / 带状态的方块 / 方块谓词 |
| `itemArg(name)` / `itemStackArg(name)` / `itemPredicateArg(name)` | 物品 / 物品堆 / 物品谓词 |
| `dimensionArg(name)` | 维度，如 `minecraft:the_nether` |
| `itemSlotArg(name)` | 物品槽位，如 `hotbar.0` |
| `particleArg(name)` | 粒子类型 |
| `colorArg(name)` | 聊天颜色名，如 `gold` |
| `identifierArg(name)` | 命名空间标识符，如 `minecraft:stone` |
| `textArgType(name)` | JSON 文本组件 |
| `nbtArg(name)` / `nbtElementArg(name)` / `nbtCompoundArg(name)` | NBT / NBT 元素 / NBT 复合标签 |

## 参数层级与 or()

每调用一次参数方法，后续内容就会**嵌套到这个参数下面一层**。想在同一层开第二个分支（比如多个子命令），用 `or` 回退：

| 方法 | 说明 |
| --- | --- |
| `or()` | 回退一层 |
| `or(argumentLevel)` | 回退到指定层级，`or(1)` 直接回到命令名本身 |
| `otherwise()` / `otherwise(argLevel)` | `or` 的同义方法（有些语言里 `or` 是关键字） |

```javascript
manager.createCommandBuilder("demo")
    .literalArg("a")            // 层级1: a
        .intArg("num")          // 层级2: a <num>
        .executes(...)
    .or(1)                      // 回到命令名
    .literalArg("b")            // 层级1: b
        .executes(...)
    .register()
// 得到 /demo a <num> 和 /demo b 两个分支
```

## executes 与 CommandContextHelper

`executes(callback)` 的回调收到一个 `CommandContextHelper`：

| 方法 | 说明 |
| --- | --- |
| `getArg(name)` | 按参数名取值（参数不存在会抛 `CommandSyntaxException`） |
| `getInput()` | 玩家输入的完整命令原文 |
| `getChild()` | 子上下文 |
| `getRange()` | 本节点匹配的字符区间 |
| `getRaw()` | 原始 Brigadier `CommandContext` |

`getArg` 声明的返回类型是 `any`，实际类型取决于参数种类，常见对应关系：

| 参数方法 | `getArg` 得到 |
| --- | --- |
| `wordArg` / `quotedStringArg` / `greedyStringArg` / `regexArgType` | 字符串 |
| `intArg` / `longArg` / `doubleArg` / `angleArg` / `timeArg` | 数字 |
| `booleanArg` | 布尔值 |
| `blockPosArg` | 位置（通常为 `BlockPosHelper`） |
| `itemArg` / `itemStackArg` | 物品（通常为 `ItemStackHelper`） |
| `textArgType` | 文本（通常为 `TextHelper`） |
| `nbtArg` 系列 | NBT（通常为 `NBTElementHelper`） |
| `colorArg` | 格式（通常为 `FormattingHelper`） |
| 其他复杂类型 | 对应 Helper 或原生对象 |

!!! tip
    拿不准类型时先 `Chat.log(ctx.getArg("xxx"))` 打印看看。`literalArg` 是固定字面量，没有值可取——要区分走了哪个子命令，给每个分支各挂一个 `executes` 即可。

!!! note "回调里别做长耗时操作 看门狗"
    官方 JSDoc 建议：如果命令处理逻辑复杂、需要 `wait` 之类的等待操作，请在回调里用 `JsMacros.runScript(file, ctx)` 把工作转交给独立脚本——`CommandContextHelper` 本身就是一个事件对象（`BaseEvent`），可以直接当第二个参数传入。

## 自动补全

### 快捷方法（挂在 CommandBuilder 上）

写在某个参数后面，为**该参数**提供补全：

| 方法 | 说明 |
| --- | --- |
| `suggestMatching(...suggestions)` / `suggestMatching(collection)` | 按输入前缀匹配一组字符串 |
| `suggestIdentifier(...ids)` / `suggestIdentifier(collection)` | 匹配一组标识符（`minecraft:xxx`） |
| `suggestBlockPositions(...positions)` / `(collection)` | 建议一组 `BlockPosHelper` 坐标 |
| `suggestPositions(...positions)` / `(collection)` | 建议 `"x y z"` 形式的坐标字符串，支持 `~`、`^` |
| `suggest(callback)` | 动态补全，回调收 `(CommandContextHelper, SuggestionsBuilderHelper)` |

### SuggestionsBuilderHelper 方法表

`suggest(callback)` 的第二个参数，用于动态生成补全：

| 方法 | 说明 |
| --- | --- |
| `getInput()` | 当前整行输入 |
| `getStart()` | 当前补全区间的起始下标 |
| `getRemaining()` / `getRemainingLowerCase()` | 光标处还没写完的部分（原样 / 小写） |
| `suggest(suggestion)` | 添加一个字符串建议 |
| `suggest(value)` | 添加一个整数建议 |
| `suggestWithTooltip(suggestion, tooltip)` | 带悬停提示的建议，`tooltip` 是 `TextHelper` |
| `suggestWithTooltip(value, tooltip)` | 整数版 |
| `suggestMatching(...)` / `suggestIdentifier(...)` / `suggestBlockPositions(...)` / `suggestPositions(...)` | 与上表同名方法一致，均有可变参数和集合两种重载 |

```javascript
.wordArg("target")
.suggest(JavaWrapper.methodToJava((ctx, builder) => {
    // 动态建议：在线玩家名（getPlayers 可能返回 null，getName 返回字符串）
    const players = World.getPlayers()
    if (players) players.forEach((p) => {
        const name = p.getName()
        if (name) builder.suggest(name)
    })
}))
```

## CommandManager 与注销命令

| 方法 | 说明 |
| --- | --- |
| `getValidCommands()` | 当前命令树中全部可用命令名列表 |
| `createCommandBuilder(name)` | 创建 `CommandBuilder` |
| `unregisterCommand(command)` | 注销命令，返回 `CommandNodeHelper`（命令不存在等情况会抛异常，记得 try/catch） |
| `reRegisterCommand(node)` | 把 `unregisterCommand` 返回的节点重新挂回去（官方注释：hacky，慎用） |
| `getArgumentAutocompleteOptions(commandPart, callback)` | 请求一段命令文本的补全选项，结果以字符串列表回调返回 |

另外 `CommandBuilder` 自己也有 `unregister()`：如果你还持有构建器对象，直接调用它即可移除该命令。

`CommandNodeHelper` 是被注销命令节点的包装（含 `fabric` 字段，对应 Fabric 侧节点），一般只用来传给 `reRegisterCommand`。

```javascript
// 重载脚本前的标准写法：先注销旧命令再注册，避免重复
const manager = Chat.getCommandManager()
try { manager.unregisterCommand("waypoint") } catch (e) {}
```

## 完整示例：/waypoint 路径点命令

支持 `add <名字>`、`list`、`del <名字>`（带自动补全），数据存在 `GlobalVars`，脚本重载后不丢失（退出游戏会清空；要永久保存请配合 [文件系统 FS](fs.md)）。

```javascript
// 建议保存为服务(Service)运行，保证命令回调的脚本上下文常驻
const manager = Chat.getCommandManager()

// 防止重复注册
try { manager.unregisterCommand("waypoint") } catch (e) {}

function loadPoints() {
    const raw = GlobalVars.getString("waypoints.data")
    return raw ? JSON.parse(raw) : {}
}

function savePoints(points) {
    GlobalVars.putString("waypoints.data", JSON.stringify(points))
}

manager.createCommandBuilder("waypoint")
    // /waypoint add <name> —— 记录当前位置
    .literalArg("add")
        .wordArg("name")
        .executes(JavaWrapper.methodToJava((ctx) => {
            const points = loadPoints()
            const name = ctx.getArg("name")
            const pos = Player.getPlayer().getPos()
            points[name] = {
                x: Math.floor(pos.x),
                y: Math.floor(pos.y),
                z: Math.floor(pos.z)
            }
            savePoints(points)
            const p = points[name]
            Chat.log(`§a已记录路径点 §e${name}§a: ${p.x}, ${p.y}, ${p.z}`)
            return true
        }))
    .or(1)
    // /waypoint list —— 列出全部路径点
    .literalArg("list")
        .executes(JavaWrapper.methodToJava((ctx) => {
            const points = loadPoints()
            const names = Object.keys(points)
            if (names.length === 0) {
                Chat.log("§7还没有路径点，用 /waypoint add <名字> 添加一个")
                return true
            }
            Chat.log(`§6===== 路径点 (${names.length}) =====`)
            names.forEach((n) => {
                const p = points[n]
                Chat.log(`§e${n} §7-> §b${p.x}, ${p.y}, ${p.z}`)
            })
            return true
        }))
    .or(1)
    // /waypoint del <name> —— 删除，参数带自动补全
    .literalArg("del")
        .wordArg("name")
        .suggest(JavaWrapper.methodToJava((ctx, builder) => {
            builder.suggestMatching(...Object.keys(loadPoints()))
        }))
        .executes(JavaWrapper.methodToJava((ctx) => {
            const points = loadPoints()
            const name = ctx.getArg("name")
            if (points[name] === undefined) {
                Chat.log(`§c没有叫 §e${name} §c的路径点`)
                return true
            }
            delete points[name]
            savePoints(points)
            Chat.log(`§a已删除路径点 §e${name}`)
            return true
        }))
    .register()

Chat.log("§a/waypoint 已注册: add <名字> | list | del <名字>")
```

试试看：

```
/waypoint add home
/waypoint list
/waypoint del home    (输入 del 后按 Tab 会补全已有名字)
```

## 注意事项

!!! warning "常见坑"
    - **重复注册**：同名命令重复 `register()` 会产生冲突或异常。重载脚本前先 `unregisterCommand`（包上 try/catch），或保留 `CommandBuilder` 对象调用 `unregister()`。
    - **回调上下文**：`executes` / `suggest` 的回调依赖注册它的脚本上下文。普通脚本运行结束上下文就被回收，命令会失效——请把注册命令的脚本作为**服务（Service）**常驻运行。
    - **贪婪参数放最后**：`greedyStringArg` 会吞掉后面的一切，它之后不能再声明参数。


