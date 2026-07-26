---
icon: lucide/monitor
---

# 客户端

`Client` 负责 Minecraft 客户端级别的能力：等待 tick、主线程执行、进入世界/连接服务器、ping 服务器、Mod 列表、注册表、剪贴板、退出游戏和网络包。

!!! note "本页对应的 API"
    本页覆盖 `Client` 命名空间的全部方法，并附带 `RegistryHelper`（注册表）与 `ServerInfoHelper`（服务器信息）两个 Helper。游戏设置相关的 `OptionsHelper` 内容较多，单独放在 [游戏设置](options.md) 页。

## Tick 等待与主线程

### waitTick

```javascript
Client.waitTick()    // 等待 1 个客户端 tick
Client.waitTick(20)  // 等待 20 个客户端 tick（约 1 秒）
```

| 签名 | 说明 |
| --- | --- |
| `waitTick(): void` | 等待 1 tick，可能抛出 `InterruptedException` |
| `waitTick(i: int): void` | 等待 `i` 个客户端 tick |

### waitTick 和 Time.sleep 的区别

两者都会阻塞当前脚本线程，但阻塞的"依据"完全不同：

| | `Client.waitTick(i)` | `Time.sleep(millis)` |
| --- | --- | --- |
| 计时单位 | 游戏 tick（客户端实际跑过的 tick） | 真实时间毫秒 |
| 游戏卡顿 / TPS 下降时 | 跟着游戏一起变慢，和游戏节奏保持同步 | 照常按真实时间醒来，可能"抢跑" |
| 游戏暂停（单人打开菜单）时 | tick 不走，脚本也停着等 | 照常醒来 |
| 适合场景 | 和游戏内行为同步：自动操作、逐 tick 轮询状态 | 和现实时间同步：定时器、请求限速、日志间隔 |

一句话：**和游戏世界打交道用 `waitTick`，和现实世界打交道用 `Time.sleep`**。写游戏内循环时优先 `waitTick`，而不是 `while(true)` 硬转或 `Time.sleep` 干等。

```javascript
// 每 tick 检查一次血量，掉血就提示
while (true) {
    const p = Player.getPlayer()
    if (p && p.getHealth() < 10) {
        Chat.log("§c血量过低！")
        Client.waitTick(100) // 提示后冷却 5 秒（游戏时间）
    }
    Client.waitTick()
}
```

!!! warning "别在主线程会等待的事件里 waitTick"
    有些事件（如 joined 模式注册的监听）本身在主线程上阻塞执行。此时再调用 `Client.waitTick()` 就是"主线程等脚本、脚本等主线程"的循环等待，会把游戏卡死。d.ts 原注释：*don't use this on an event that the main thread waits on (joins)... that'll cause circular waiting*。

### runOnMainThread

把任务丢到 Minecraft 主线程执行。很多 Minecraft 内部操作（打开界面、直接改游戏对象等）只允许在主线程做，脚本线程直接调用会崩溃或行为异常，这时就需要它。

| 签名 | 说明 |
| --- | --- |
| `runOnMainThread(runnable): void` | 在主线程执行任务 |
| `runOnMainThread(runnable, watchdogMaxTime: long): void` | 附带看门狗超时（毫秒） |
| `runOnMainThread(runnable, await: boolean, watchdogMaxTime: long): void` | `await` 为 `true` 时阻塞脚本直到任务完成 |

参数说明：

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `runnable` | `MethodWrapper` | 用 `JavaWrapper.methodToJava(...)` 包装的回调 |
| `watchdogMaxTime` | `long` | 看门狗允许任务占用主线程的最长毫秒数，超时会杀掉脚本 |
| `await` | `boolean` | 是否等任务在主线程跑完再继续脚本 |

```javascript
Client.runOnMainThread(JavaWrapper.methodToJava(() => {
    Chat.log("这段在主线程执行")
}))
```

!!! warning "看门狗风险：主线程上别干重活"
    你塞进 `runOnMainThread` 的代码是直接占用游戏主线程的——游戏画面在这期间是冻结的。任务太慢就是肉眼可见的卡顿；超过看门狗时限（`watchdogMaxTime`），脚本会被直接杀死。只把**必须**在主线程做的一小段 Minecraft 操作放进去，循环、休眠、网络请求一律留在脚本线程。绝对不要在主线程任务里调用 `Client.waitTick()` 或 `Time.sleep()`。

## 客户端状态与信息

```javascript
Chat.log(Client.mcVersion())      // 例如 "1.21.1"
Chat.log(Client.getFPS())         // Minecraft 的 FPS 调试字符串
Chat.log(Client.getModLoader())   // Mod 加载器名称
Chat.log(Client.isDevEnv())       // 是否开发环境
```

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `mcVersion()` | `string` | 当前 Minecraft 版本 |
| `getFPS()` | `string` | FPS 调试字符串 |
| `getGameOptions()` | [`OptionsHelper`](options.md) | 游戏设置（视频、声音、按键、聊天等），详见 [游戏设置](options.md) |
| `getMinecraft()` | `any` | 原始 Minecraft 客户端对象（混淆名，可配合 [Minecraft Mappings Viewer](https://wagyourtail.xyz/Projects/Minecraft%20Mappings%20Viewer/App) 使用） |
| `getModLoader()` | `string` | Mod 加载器名称 |
| `isDevEnv()` | `boolean` | 是否运行在开发环境 |
| `grabMouse()` | `void` | 让游戏认为鼠标在窗口内（会自动关闭"失去焦点时暂停"），适合后台挂机脚本 |

## 进入世界与连接服务器

| 方法 | 说明 |
| --- | --- |
| `loadWorld(folderName: string)` | 进入单人存档，参数是存档文件夹名 |
| `connect(ip: string)` | 连接服务器（默认端口） |
| `connect(ip: string, port: int)` | 连接服务器并指定端口 |
| `disconnect()` | 断开当前连接 |
| `disconnect(callback)` | 断开连接，完成后回调；回调收到一个 `boolean` |

```javascript
Client.loadWorld("我的世界存档")        // 单人
Client.connect("example.com")          // 多人
Client.connect("127.0.0.1", 25565)

Client.disconnect(JavaWrapper.methodToJava((success) => {
    Chat.log(`断开完成: ${success}`)
}))
```

!!! tip "查当前连接的服务器地址"
    这个功能在 `World` 命名空间：`World.getCurrentServerAddress()` 返回 `server.address/server.ip:port` 形式的字符串，没连接时为 `null`。见 [世界与方块](world.md)。

## Ping 服务器

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `ping(ip: string)` | [`ServerInfoHelper`](#serverinfohelper) | 同步 ping，可能抛 `UnknownHostException` |
| `pingAsync(ip, callback)` | `void` | 异步 ping，回调收到 `(info, err)`：成功时 `info` 是 `ServerInfoHelper`，失败时 `err` 是 `IOException` |
| `cancelAllPings()` | `void` | 取消所有进行中的 ping |

```javascript
// 同步：会阻塞脚本线程直到有结果
const info = Client.ping("example.com")
Chat.log(`${info.getLabel().getString()} 延迟 ${info.getPing()}ms`)

// 异步：不阻塞，结果回调
Client.pingAsync("example.com", JavaWrapper.methodToJava((info, err) => {
    if (err) Chat.log(`ping 失败: ${err}`)
    else Chat.log(`在线 ${info.getPlayerCountLabel().getString()}，延迟 ${info.getPing()}ms`)
}))
```

### ServerInfoHelper

`ping` 的返回值，包装了服务器列表里那种信息。返回 `TextHelper` 的方法可以用 `.getString()` 取纯文本，见 [常用类型](types.md#texthelper)。

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getName()` | `string` | 服务器名 |
| `getAddress()` | `string` | 服务器地址 |
| `getPing()` | `number` | 延迟（毫秒） |
| `getLabel()` | [`TextHelper`](types.md#texthelper) | MOTD 描述文本 |
| `getPlayerCountLabel()` | [`TextHelper`](types.md#texthelper) | 在线人数文本（如 `12/100`） |
| `getVersion()` | [`TextHelper`](types.md#texthelper) | 版本文本 |
| `getProtocolVersion()` | `number` | 协议版本号 |
| `getPlayerListSummary()` | `JavaList<TextHelper>` | 玩家列表摘要（悬停服务器条目时那几行） |
| `resourcePackPolicy()` | `string` | 服务器资源包策略 |
| `getIcon()` | `JavaArray<number>` | 服务器图标的字节数组 |
| `isOnline()` | `boolean` | 是否在线 |
| `isLocal()` | `boolean` | 是否局域网服务器 |
| `getNbt()` | `NBTElementHelper$NBTCompoundHelper` | 原始 NBT 数据 |

## Mod 信息

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isModLoaded(modId: string)` | `boolean` | 指定 Mod 是否已加载 |
| `getLoadedMods()` | `JavaList<ModContainerHelper>` | 所有已加载 Mod |
| `getMod(modId: string)` | `ModContainerHelper \| null` | 指定 Mod 的容器，未加载返回 `null` |

```javascript
if (Client.isModLoaded("jsmacros")) {
    Chat.log("JsMacros 已加载")
}

const mods = Client.getLoadedMods()
Chat.log(`已加载 Mod 数量: ${mods.size()}`)
```

## 注册表

### 快捷方法

```javascript
const items = Client.getRegisteredItems()
const blocks = Client.getRegisteredBlocks()
Chat.log(`${items.size()} 种物品, ${blocks.size()} 种方块`)
```

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getRegisteredItems()` | `JavaList<`[`ItemHelper`](inventory.md)`>` | 所有已注册物品 |
| `getRegisteredBlocks()` | `JavaList<`[`BlockHelper`](blocks.md)`>` | 所有已注册方块 |
| `getRegistryManager()` | [`RegistryHelper`](#registryhelper) | 完整注册表访问 |

### RegistryHelper

`Client.getRegistryManager()` 返回的 helper，可以按 id 查任何注册表内容。id 支持省略命名空间（`"diamond"` 等价于 `"minecraft:diamond"`）。

```javascript
const reg = Client.getRegistryManager()
const item = reg.getItem("diamond_sword")   // 等价于 "minecraft:diamond_sword"
Chat.log(item.getName().getString())

// 列出所有附魔 id
reg.getEnchantmentIds().forEach(id => Chat.log(id))
```

按 id 取单个对象：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getItem(id)` | [`ItemHelper`](inventory.md) | 按 id 取物品 |
| `getItemStack(id)` | [`ItemStackHelper`](inventory.md) | 按 id 取物品栈 |
| `getItemStack(id, nbt)` | [`ItemStackHelper`](inventory.md) | 附带 NBT，NBT 非法时抛异常 |
| `getBlock(id)` | [`BlockHelper`](blocks.md) | 按 id 取方块 |
| `getBlockState(id)` | [`BlockStateHelper`](blocks.md) | 按 id 取方块状态 |
| `getBlockState(id, nbt)` | [`BlockStateHelper`](blocks.md) | 附带 NBT |
| `getEnchantment(id)` | [`EnchantmentHelper`](inventory.md) | 按 id 取附魔 |
| `getEnchantment(id, level)` | [`EnchantmentHelper`](inventory.md) | 指定等级 |
| `getStatusEffect(id)` | `StatusEffectHelper` | 按 id 取状态效果（时长为 0） |
| `getStatusEffects()` | `JavaList<StatusEffectHelper>` | 所有状态效果 |
| `getEntity(type)` | [`EntityHelper`](entities.md) | 按 id 创建实体 helper |
| `getRawEntityType(type)` | `any` | 原始 `EntityType` 对象 |
| `getFluidState(id)` | `FluidStateHelper` | 按 id 取流体状态 |
| `getIdentifier(identifier)` | `any` | 原始 Minecraft `Identifier` |
| `getItems()` / `getBlocks()` / `getEnchantments()` | `JavaList<...>` | 对应类型的完整列表 |

各类 id 列表（全部返回 `JavaList<string>` 形式的 id）：

| 方法 | 内容 |
| --- | --- |
| `getItemIds()` / `getBlockIds()` | 物品 / 方块 |
| `getEnchantmentIds()` / `getPotionTypeIds()` | 附魔 / 药水 |
| `getEntityTypeIds()` / `getStatusEffectIds()` | 实体类型 / 状态效果 |
| `getFeatureIds()` / `getStructureFeatureIds()` | 地物 / 结构 |
| `getPaintingIds()` / `getParticleTypeIds()` | 画 / 粒子类型 |
| `getGameEventNames()` / `getStatTypeIds()` | 游戏事件 / 统计类型 |
| `getBlockEntityTypeIds()` / `getScreenHandlerIds()` | 方块实体 / 界面处理器 |
| `getRecipeTypeIds()` / `getEntityAttributeIds()` | 配方类型 / 实体属性 |
| `getVillagerTypeIds()` / `getVillagerProfessionIds()` | 村民类型 / 村民职业 |
| `getPointOfInterestTypeIds()` / `getMemoryModuleTypeIds()` | 兴趣点 / 记忆模块 |
| `getSensorTypeIds()` / `getActivityTypeIds()` | 感知器 / 活动类型 |

静态方法（写高级 NBT/序列化代码才会用到）：

| 方法 | 说明 |
| --- | --- |
| `RegistryHelper.getNbtOps()` | 带注册表信息的 NBT ops，编码附魔、纹饰等注册表内容时代替 `NbtOps.INSTANCE` |
| `RegistryHelper.getWrapperLookup()` | 真实的 `HolderLookup.Provider`，供需要 "WrapperLookup" 的 API 使用 |
| `RegistryHelper.parseIdentifier(id)` | 字符串转原始 `Identifier` |
| `RegistryHelper.parseNameSpace(id)` | 取出 id 的命名空间部分 |

## 剪贴板

| 方法 | 说明 |
| --- | --- |
| `getClipboard(): string` | 读取系统剪贴板 |
| `setClipboard(text: string)` | 写入系统剪贴板 |

```javascript
Client.setClipboard("复制到剪贴板")
Chat.log(Client.getClipboard())
```

## 退出游戏

| 方法 | 说明 |
| --- | --- |
| `exitGamePeacefully(): void` | 尝试和平退出游戏 |
| `exitGameForcefully(): never` | 强制退出游戏 |
| `shutdown(): never` | 关闭客户端并等待游戏停止，之后不再执行任何代码 |

```javascript
Client.exitGamePeacefully()
// Client.exitGameForcefully()
// Client.shutdown()
```

!!! warning "shutdown 不等你的线程"
    `shutdown()` 会等游戏停止，但不会等待脚本里 join 的其它线程，脚本可能停在一个不确定的位置。`exitGameForcefully()` 和 `shutdown()` 之后的代码永远不会执行。

## 网络包（高级）

| 方法 | 说明 |
| --- | --- |
| `sendPacket(packet)` | 发送一个原始 Minecraft packet |
| `receivePacket(packet)` | 让客户端"收到"一个 packet |
| `createPacketByteBuffer()` | 返回 `PacketByteBufferHelper`，用于构造和修改 packet |

!!! warning "高级危险区"
    网络包 API 很容易触发反作弊或造成客户端状态不同步。除非你清楚包结构和服务端规则，否则不要在多人服务器里使用。

## 生成 TypeScript 类型文件

给编辑器补全用的 `McIdsAndEnums.d.ts`（包含物品/方块 id、枚举等字面量类型）可以用脚本自己生成：

| 签名 | 说明 |
| --- | --- |
| `generateTypescriptIdsAndEnums(): string` | 生成内容并作为字符串返回 |
| `generateTypescriptIdsAndEnums(full: boolean): string` | `full` 为 `true` 时额外扫描 jar 生成屏幕类（用 ASM，较慢） |
| `generateTypescriptIdsAndEnums(file): void` | 直接写入 `FS.open()` 打开的文件 |
| `generateTypescriptIdsAndEnums(file, full: boolean): void` | 写入文件并附带屏幕类 |

```javascript
const file = FS.open("./McIdsAndEnums.d.ts")
file.write("") // 该方法不会自动清空文件，先手动清空
Client.generateTypescriptIdsAndEnums(file)
Chat.log("已生成 McIdsAndEnums.d.ts")
```
