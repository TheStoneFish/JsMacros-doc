---
icon: lucide/radio
---

# 事件系统

事件系统让脚本在聊天、按键、Tick、打开容器、收发包等时机自动运行。核心入口是 `JsMacros.on`、`JsMacros.once`、`JsMacros.off` 和 `JsMacros.waitForEvent`。

## 注册监听器

```javascript
JsMacros.on("Tick", JavaWrapper.methodToJava((e, context) => {
    // 每 tick 执行一次
}))
```

只触发一次：

```javascript
JsMacros.once("JoinServer", JavaWrapper.methodToJava((e) => {
    Chat.log(`进入服务器: ${e.address}`)
}))
```

取消监听：

```javascript
const listener = JsMacros.on("Tick", JavaWrapper.methodToJava(() => {
    Chat.log("tick")
}))

JsMacros.off(listener)
```

## joined 参数

`JsMacros.on(event, joined, callback)` 的第二个参数控制回调是否阻塞触发事件的线程。

| joined | 行为 | 适合场景 |
| --- | --- | --- |
| `false` | 回调异步运行，不阻塞游戏线程 | Tick 统计、HUD 更新、普通提示 |
| `true` | 回调加入事件线程，可修改/取消部分事件 | `SendMessage`、`RecvMessage`、`ClickSlot`、`DropSlot` |

!!! warning "看门狗"
    joined 回调不要长时间阻塞。需要先取消或修改事件，再调用 `context.releaseLock()`，然后才做耗时工作。

```javascript
JsMacros.on("SendMessage", true, JavaWrapper.methodToJava((e, context) => {
    if (e.message.startsWith(".local")) {
        e.cancel()
        context.releaseLock()
        Chat.log("这条消息只在本地处理")
    }
}))
```

## 等待事件

`JsMacros.waitForEvent` 会阻塞当前脚本，直到事件发生。

```javascript
Chat.log("请按任意键")
const result = JsMacros.waitForEvent("Key")
Chat.log(`按键事件: ${result.event.key}`)
```

可以传 `join` 和过滤器：

```javascript
const result = JsMacros.waitForEvent(
    "RecvMessage",
    false,
    JavaWrapper.methodToJava((e) => e.text.getString().includes("完成"))
)
Chat.log(result.event.text.getString())
```

## 常用事件表

| 事件 | 触发时机 | 常用字段/能力 |
| --- | --- | --- |
| `Tick` | 每游戏 tick | 周期任务、状态轮询 |
| `Key` | 键盘输入 | 可取消；适合自定义快捷键 |
| `MouseScroll` | 鼠标滚轮 | 可取消；适合滚轮菜单 |
| `SendMessage` | 玩家发送聊天/命令前 | 可取消或改写消息 |
| `RecvMessage` | 客户端收到聊天消息 | 可取消；适合聊天过滤 |
| `OpenScreen` | 打开任意屏幕 | 读取屏幕信息 |
| `OpenContainer` | 打开容器前 | 可取消；适合自动处理容器 |
| `ClickSlot` | 点击槽位 | 可取消；适合保护物品 |
| `DropSlot` | 丢物品 | 可取消；适合防误丢 |
| `SlotUpdate` | 槽位更新 | 背包/容器监听 |
| `InteractBlock` | 与方块交互 | 记录坐标和方向 |
| `InteractEntity` | 与实体交互 | 读取目标实体 |
| `AttackBlock` | 攻击方块 | 挖掘相关 |
| `AttackEntity` | 攻击实体 | 战斗相关 |
| `EntityLoad` / `EntityUnload` | 实体加载/卸载 | 实体缓存 |
| `PlayerJoin` / `PlayerLeave` | 玩家加入/离开 | 玩家列表提示 |
| `JoinServer` / `Disconnect` | 服务器连接变化 | 自动初始化/清理 |
| `RecvPacket` / `SendPacket` | 网络包收发 | 高风险高级用法 |
| `Title` | 标题/副标题/ActionBar | 可取消 |
| `Sound` / `ServerSound` | 声音播放 | 可取消 |
| `Particle` | 粒子生成 | 可取消 |
| `Custom` | 自定义事件 | 脚本间通信 |

完整事件名以 `JsMacros-2.1.0.d.ts` 的 `interface Events` 为准。

## 自定义事件

```javascript
const ev = JsMacros.createCustomEvent("MyEvent")
ev.putString("from", "script-a")
ev.trigger()
```

监听：

```javascript
JsMacros.on("MyEvent", JavaWrapper.methodToJava((e) => {
    Chat.log(e.getString("from"))
}))
```

自定义事件里可以存：

| 方法 | 读取 |
| --- | --- |
| `putInt(name, value)` | `getInt(name)` |
| `putString(name, value)` | `getString(name)` |
| `putDouble(name, value)` | `getDouble(name)` |
| `putBoolean(name, value)` | `getBoolean(name)` |
| `putObject(name, value)` | `getObject(name)` |

## 事件监听清理

脚本重载或服务停止时，记得清理监听器。

```javascript
const listener = JsMacros.on("Tick", JavaWrapper.methodToJava(() => {}))
context.unregisterOnStop(true, listener)
```

也可以按范围关闭：

```javascript
JsMacros.disableScriptListeners()
JsMacros.disableAllListeners("RecvMessage")
```

