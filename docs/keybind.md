---
icon: lucide/keyboard
---

# 键盘与鼠标输入

`KeyBind` 是处理输入的全局对象，负责两件事：

1. **模拟输入**：让脚本"替你"按下键盘按键或鼠标按钮；
2. **读取和修改按键绑定**：查询、更改 Minecraft 设置里的按键绑定。

配合 `Key` 事件（见[事件参考](events.md)），还可以把任意按键变成自定义快捷键。

## 方法总览

| 方法 | 说明 | 版本 |
| --- | --- | --- |
| `key(keyName, keyState)` | 设置某个物理键的按下状态 | - |
| `pressKey(keyName)` | 按下物理键，等价于 `key(keyName, true)` | 1.8.4 |
| `releaseKey(keyName)` | 松开物理键，等价于 `key(keyName, false)` | 1.8.4 |
| `keyBind(keyBind, keyState)` | 设置某个游戏按键绑定的按下状态 | 1.2.2 |
| `pressKeyBind(keyBind)` | 按下绑定，等价于 `keyBind(keyBind, true)` | 1.8.4 |
| `releaseKeyBind(keyBind)` | 松开绑定，等价于 `keyBind(keyBind, false)` | 1.8.4 |
| `getPressedKeys()` | 返回当前按下的所有键（`JavaSet<Key>`） | 1.2.6 |
| `getKeyBindings()` | 返回所有按键绑定的 `JavaMap<Bind, Key>` | 1.2.2 |
| `setKeyBind(bind, key)` | 修改一个按键绑定，`key` 传 `null` 可清空 | 1.2.2 |
| `getKeyCode(keyName)` | 返回原生 Minecraft 按键对象（官方注释：不建议使用） | - |

## 按键名称体系

Minecraft 用统一的字符串命名每个物理键：键盘键是 `key.keyboard.xxx`，鼠标键是 `key.mouse.xxx`。`KeyBind` 的所有"物理键"参数、`getPressedKeys()` 的返回值、`Key` 事件的 `event.key` 用的都是这一套名称。

### 常用键名对照表

| 分类 | 键名 |
| --- | --- |
| 字母 | `key.keyboard.a` ~ `key.keyboard.z` |
| 主键盘数字 | `key.keyboard.0` ~ `key.keyboard.9` |
| F 键 | `key.keyboard.f1` ~ `key.keyboard.f12` |
| 方向键 | `key.keyboard.up` / `key.keyboard.down` / `key.keyboard.left` / `key.keyboard.right` |
| 空格 / 回车 / Tab | `key.keyboard.space` / `key.keyboard.enter` / `key.keyboard.tab` |
| Esc / 退格 | `key.keyboard.escape` / `key.keyboard.backspace` |
| Shift | `key.keyboard.left.shift` / `key.keyboard.right.shift` |
| Ctrl | `key.keyboard.left.control` / `key.keyboard.right.control` |
| Alt | `key.keyboard.left.alt` / `key.keyboard.right.alt` |
| Insert / Delete | `key.keyboard.insert` / `key.keyboard.delete` |
| Home / End / 翻页 | `key.keyboard.home` / `key.keyboard.end` / `key.keyboard.page.up` / `key.keyboard.page.down` |
| 小键盘 | `key.keyboard.keypad.0` ~ `keypad.9`、`keypad.enter`、`keypad.add`、`keypad.subtract` 等 |
| 常用符号 | `key.keyboard.comma`(,)、`period`(.)、`slash`(/)、`semicolon`(;)、`minus`(-)、`equal`(=)、`grave.accent`(反引号) |
| 鼠标左 / 右 / 中键 | `key.mouse.left` / `key.mouse.right` / `key.mouse.middle` |
| 鼠标侧键 | `key.mouse.4`、`key.mouse.5`（更多侧键依次类推） |
| 未绑定 | `key.keyboard.unknown` |

!!! tip "不确定键名怎么办"
    在脚本里执行下面这行，然后按一下想查的键，聊天栏就会显示它的名称：

    ```javascript
    Chat.log(JsMacros.waitForEvent("Key").event.key)
    ```

### 修饰键组合 KeyMods

`Key` 事件的 `event.mods` 字段表示触发时按住的修饰键。按 d.ts 中 `KeyMod` / `KeyMods` 的定义，修饰键统一用左侧键名表示，多个修饰键按 **shift → ctrl → alt** 的顺序用 `+` 连接：

| 组合 | mods 值 |
| --- | --- |
| Shift | `key.keyboard.left.shift` |
| Ctrl | `key.keyboard.left.control` |
| Alt | `key.keyboard.left.alt` |
| Shift+Ctrl | `key.keyboard.left.shift+key.keyboard.left.control` |
| Ctrl+Alt | `key.keyboard.left.control+key.keyboard.left.alt` |
| Shift+Ctrl+Alt | `key.keyboard.left.shift+key.keyboard.left.control+key.keyboard.left.alt` |

## key 和 keyBind 的区别

`KeyBind` 里有两类"模拟按键"方法，操作对象不同：

| 方法 | 操作对象 |
| --- | --- |
| `key("key.keyboard.w", true)` | 直接按某个物理键 |
| `keyBind("key.forward", true)` | 按 Minecraft 绑定项 |

如果玩家改过按键，`keyBind` 会跟随设置；`key` 不会。官方 JSDoc 对 `keyBind` 的原话是："This is probably the one you should use."（你多半应该用这个）——想让角色做"前进 / 攻击 / 跳跃"这类游戏动作时，优先用 `keyBind` 系列，脚本不会因为玩家改键而失效。

## 模拟物理按键

`pressKey` / `releaseKey`（或底层的 `key(keyName, keyState)`）直接设置物理键的按下状态，键盘和鼠标通用：

```javascript
KeyBind.pressKey("key.keyboard.space")
Client.waitTick(2)
KeyBind.releaseKey("key.keyboard.space")
```

快捷写法（`key` 一个方法管按下和松开）：

```javascript
KeyBind.key("key.keyboard.w", true)
Client.waitTick(20)
KeyBind.key("key.keyboard.w", false)
```

模拟鼠标也一样，比如按住右键吃东西：

```javascript
KeyBind.pressKey("key.mouse.right")
Client.waitTick(40)
KeyBind.releaseKey("key.mouse.right")
```

!!! warning "按下的键不会自己松开"
    `pressKey` / `pressKeyBind` 只是把状态置为"按下"，**不调用对应的 release，这个键会一直保持按住**。如果脚本在按下和松开之间报错退出，角色可能一直往前跑或一直挥手——需要手动按一下再松开那个真实按键才能解除。重要操作建议用 `try / finally` 保证松开：

    ```javascript
    KeyBind.pressKeyBind("key.forward")
    try {
        Client.waitTick(100)
    } finally {
        KeyBind.releaseKeyBind("key.forward")
    }
    ```

!!! tip "按下和松开之间等一两 tick"
    同一瞬间连续调用 press 和 release，游戏可能来不及把它当成一次完整点击处理。想模拟"点一下"，在中间加 `Client.waitTick(1)` 更稳。

## 触发游戏内按键绑定

`pressKeyBind` / `releaseKeyBind`（或 `keyBind(bind, keyState)`）按"绑定名"操作，跟随玩家的按键设置：

```javascript
KeyBind.pressKeyBind("key.forward")
Client.waitTick(20)
KeyBind.releaseKeyBind("key.forward")
```

常见绑定名：

| 绑定 | 含义 |
| --- | --- |
| `key.forward` | 前进 |
| `key.back` | 后退 |
| `key.left` | 向左 |
| `key.right` | 向右 |
| `key.jump` | 跳跃 |
| `key.sneak` | 潜行 |
| `key.sprint` | 疾跑 |
| `key.attack` | 攻击 |
| `key.use` | 使用 |
| `key.inventory` | 背包 |
| `key.drop` | 丢弃物品 |
| `key.swapOffhand` | 与副手交换 |
| `key.pickItem` | 选取方块 |
| `key.chat` | 打开聊天 |
| `key.screenshot` | 截图 |
| `key.togglePerspective` | 切换视角 |
| `key.hotbar.1` ~ `key.hotbar.9` | 快捷栏 1~9 |

完整列表可以直接打印 `getKeyBindings()`，见下文。

## 查询当前按下的键

`getPressedKeys()` 返回一个 Java `Set`，包含此刻所有按下的键（含鼠标键），用 `contains` 判断：

```javascript
const keys = KeyBind.getPressedKeys()
if (keys.contains("key.keyboard.left.shift")) {
    Chat.log("按住了 Shift")
}
```

也可以遍历打印，看看这一刻到底按了什么：

```javascript
for (const k of KeyBind.getPressedKeys()) {
    Chat.log(k)
}
```

## 读取与修改按键绑定

`getKeyBindings()` 返回"绑定名 → 键名"的 `JavaMap`，可以查询也可以遍历：

```javascript
const binds = KeyBind.getKeyBindings()
Chat.log(`跳跃绑定在: ${binds.get("key.jump")}`)

// 打印全部绑定，顺便能查到所有可用的绑定名
for (const bind of binds.keySet()) {
    Chat.log(`${bind} -> ${binds.get(bind)}`)
}
```

`setKeyBind(bind, key)` 修改绑定，效果和在设置界面改键一样：

```javascript
const old = KeyBind.getKeyBindings().get("key.jump")
KeyBind.setKeyBind("key.jump", "key.keyboard.g")
Chat.log(`原跳跃键: ${old}`)
```

传 `null` 可以清空绑定：

```javascript
KeyBind.setKeyBind("key.sprint", null)
```

!!! warning "记得恢复"
    修改玩家按键设置会影响正常游戏。临时修改时请保存旧值，并在脚本结束时恢复。

## 配合 Key 事件：自定义快捷键

`Key` 事件在每次键盘/鼠标按键变化时触发，字段如下（完整事件列表见[事件参考](events.md)）：

| 字段 | 类型 | 含义 |
| --- | --- | --- |
| `action` | number | `1` 按下，`0` 松开 |
| `key` | string | 键名，如 `key.keyboard.g` |
| `mods` | string | 修饰键组合，见上文[修饰键组合 KeyMods](#keymods) |

最简单的自定义快捷键——按 G 打招呼：

```javascript
JsMacros.on("Key", JavaWrapper.methodToJava((e) => {
    if (e.action === 1 && e.key === "key.keyboard.g") {
        Chat.log("按下了 G")
    }
}))
```

`Key` 事件是可取消的：用 `joined = true` 注册并调用 `e.cancel()`，可以"吞掉"这次按键，不让游戏处理。比如把 Ctrl+B 做成纯脚本快捷键：

```javascript
JsMacros.on("Key", true, JavaWrapper.methodToJava((e, context) => {
    if (e.action === 1 && e.key === "key.keyboard.b"
        && e.mods === "key.keyboard.left.control") {
        e.cancel()
        context.releaseLock()
        Chat.log("触发了 Ctrl+B 快捷键")
    }
}))
```

!!! note "鼠标滚轮不走 Key 事件"
    滚轮有单独的 `MouseScroll` 事件（`deltaX` / `deltaY`，同样可取消），见[事件参考](events.md)。

## 经典示例：按住 R 键连点攻击

综合运用 `getPressedKeys` 和 `pressKeyBind` / `releaseKeyBind`。脚本运行期间，按住 R 就会以玩家设置的攻击键连点，松开即停：

```javascript
// 建议注册为服务运行（见"服务"页），也可手动运行、在 JsMacros 界面停止
const TRIGGER = "key.keyboard.r"   // 触发键
const INTERVAL = 4                 // 每次点击间隔（tick）

while (true) {
    if (KeyBind.getPressedKeys().contains(TRIGGER)) {
        KeyBind.pressKeyBind("key.attack")
        Client.waitTick(1)
        KeyBind.releaseKeyBind("key.attack")
        Client.waitTick(INTERVAL)
    } else {
        Client.waitTick(5)
    }
}
```

把 `"key.attack"` 换成 `"key.use"` 就是连点右键；改 `INTERVAL` 可以控制点击速度。长期使用建议做成[服务](services.md)。

## getKeyCode：基本用不到

`getKeyCode(keyName)` 返回原生 Minecraft 的按键对象（`InputConstants$Key`），官方 JSDoc 原话是 "Dont use this one..."。只有在和 Java 层 API 交互时才可能用到，日常脚本请直接用上面的字符串键名。

!!! warning "多人服务器慎用自动化输入"
    连点、自动移动这类模拟输入在部分服务器属于违规行为，可能被反作弊判定。请先确认服务器规则，单人或允许的场合再使用。
