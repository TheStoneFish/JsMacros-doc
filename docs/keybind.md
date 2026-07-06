---
icon: lucide/keyboard
---

# 键盘输入

`KeyBind` 可以模拟按键，也可以读取和修改 Minecraft 原版按键绑定。

## key 和 keyBind 的区别

| 方法 | 操作对象 |
| --- | --- |
| `key("key.keyboard.w", true)` | 直接按某个物理键 |
| `keyBind("key.forward", true)` | 按 Minecraft 绑定项 |

如果玩家改过按键，`keyBind` 会跟随设置；`key` 不会。

## 模拟按键

```javascript
KeyBind.pressKey("key.keyboard.space")
Client.waitTick(2)
KeyBind.releaseKey("key.keyboard.space")
```

快捷写法：

```javascript
KeyBind.key("key.keyboard.w", true)
Client.waitTick(20)
KeyBind.key("key.keyboard.w", false)
```

## 操作游戏按键绑定

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

## 读取当前按键

```javascript
const keys = KeyBind.getPressedKeys()
if (keys.contains("key.keyboard.left.shift")) {
    Chat.log("按住了 Shift")
}
```

## 修改按键绑定

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

