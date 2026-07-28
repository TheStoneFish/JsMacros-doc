---
icon: lucide/binary
---

# 数据包

`RecvPacket` / `SendPacket` 事件加上 `PacketByteBufferHelper`，可以监听、取消、修改乃至伪造
Minecraft 客户端与服务器之间的网络数据包。这是 JsMacros 里最底层的能力之一。

!!! danger "高级功能，风险自负"
    - **可能被封号**：随意向服务器发包、修改发出的包，在插件服/小游戏服务器上很容易被
      反作弊系统判定为作弊客户端，轻则踢出重则封号。
    - **可能状态不同步**：取消或篡改包会让客户端和服务器对世界的认知不一致，出现"幽灵方块"、
      背包错乱、位置回弹等问题，严重时直接掉线。
    - 包的名称、字段顺序随游戏版本变化，本页示例中的包名和字段格式**一切以实测为准**。
    - 新手请先把[事件系统](events.md)和普通 API 用熟，再来碰这一层。

## 监听收发包

两个事件（对照 d.ts `Events` 命名空间核实）：

| 事件 | 成员 | 说明 |
| --- | --- | --- |
| `RecvPacket` | `packet` | 原始包对象，可读可写（可整个替换），可能为 `null` |
| | `type`（只读） | 包名字符串，如 `"BlockUpdateS2CPacket"` |
| | `getPacketBuffer()` | 返回 `PacketByteBufferHelper`，用于读取/修改包数据 |
| | `cancel()` | 取消这个包（需要 joined 监听） |
| `SendPacket` | `packet` / `type` / `getPacketBuffer()` / `cancel()` | 同上 |
| | `replacePacket(...args)` | 用给定构造参数替换成同类型新包；`packet` 为 `null` 时抛异常。官方注释建议优先用 `getPacketBuffer()` 修改 |

按 `type` 过滤的基本示例：

```javascript
JsMacros.on("RecvPacket", JavaWrapper.methodToJava((e) => {
    // 包名以 getPacketNames() / 实际日志为准
    if (e.type === "BlockUpdateS2CPacket") {
        Chat.log("收到一个方块更新包")
    }
}))
```

!!! warning "收发包事件的频率极高"
    每秒会触发成百上千次，回调里别做重活，更别无条件 `Chat.log`（会刷屏卡死聊天栏）。
    强烈建议配合[事件过滤器](event_filters.md)在 Java 侧先按 `type` 过滤，
    d.ts 里官方给的例子就是 `compile('RecvPacket', 'eq(event.type, "BlockUpdateS2CPacket")')`。

### 怎么知道有哪些包名

`getPacketNames()` 返回全部包名列表（官方注释：这些名字只是为了方便访问，将来可能变化）：

```javascript
const names = Client.createPacketByteBuffer().getPacketNames()
Chat.log(`共 ${names.size()} 个包名`)

// 找出名字里带 "Chat" 的包
for (let i = 0; i < names.size(); i++) {
    const name = names.get(i)
    if (String(name).includes("Chat")) Chat.log(name)
}
```

另外还有静态方法 `PacketByteBufferHelper.getPacketName(packet)` 可以从包对象反查名字：

```javascript
const PBBH = Packages.xyz.wagyourtail.jsmacros.client.api.helper.PacketByteBufferHelper

JsMacros.on("SendPacket", JavaWrapper.methodToJava((e) => {
    // 与 e.type 等价的另一种写法
    Chat.log(PBBH.getPacketName(e.packet))
}))
```

## 取消包

取消/修改事件需要 joined 监听（第二个参数传 `true`，详见[事件系统](events.md)）：

```javascript
// 示例：拦下客户端发出的某类包（包名以实测为准）
JsMacros.on("SendPacket", true, JavaWrapper.methodToJava((e, context) => {
    if (e.type === "HandSwingC2SPacket") {
        e.cancel()   // 服务器将收不到这个包
    }
    context.releaseLock()
}))
```

!!! warning "取消包 ≠ 取消行为"
    取消 `SendPacket` 只是不把包发出去，客户端本地该发生的还是发生了；
    取消 `RecvPacket` 只是客户端装作没收到，服务器那边状态照旧。两边不同步的后果自己承担。

## 读取包内容：PacketByteBufferHelper

`event.getPacketBuffer()` 会把包序列化成一个字节缓冲区并包上 helper。数据包没有"字段名"，
只有**按固定顺序排列的二进制数据**，所以你必须按包的实际序列化顺序调用对应的 `readXxx` 方法。

```javascript
JsMacros.on("RecvPacket", JavaWrapper.methodToJava((e) => {
    if (e.type !== "BlockUpdateS2CPacket") return

    const buf = e.getPacketBuffer()
    // 字段顺序必须与该包的序列化格式一致（以实测/反编译为准）
    const pos = buf.readBlockPos()
    Chat.log(`方块更新: ${pos.getX()}, ${pos.getY()}, ${pos.getZ()}`)
}))
```

### 读写的几个坑

- **读取会移动指针**：每次 `readXxx` 都从当前 `readerIndex` 往后读，读错类型后面就全错位。
- **读完要复位**：`reset()` 会把缓冲区恢复到 helper 创建时的状态（官方注释原话）。
  读了一半还想重读、或者读完之后还要 `toPacket()`，先 `reset()`。
  也可以用 `markReaderIndex()` / `resetReaderIndex()` 手动做书签。
- **修改缓冲区不会自动生效**：官方注释明确说，改完缓冲区要用 `toPacket()` 拿到修改后的包，
  再赋回 `event.packet` 替换原包。

修改并替换包的完整流程（joined 监听）：

```javascript
JsMacros.on("SendPacket", true, JavaWrapper.methodToJava((e, context) => {
    if (e.type === "某个包名") {
        const buf = e.getPacketBuffer()
        // ...按顺序 readXxx / setXxx / writeXxx 修改...
        e.packet = buf.toPacket()   // 用修改后的缓冲区重建包并替换
    }
    context.releaseLock()
}))
```

## 构造并发送自定义包

`Client.createPacketByteBuffer()` 返回一个空缓冲区 helper（d.ts 原文：a helper to modify and
send minecraft packets）。流程：写入字段 → `sendPacket(包名)`。

```javascript
const buf = Client.createPacketByteBuffer()

// 示例：挥手包只有一个表示用哪只手的枚举字段（VarInt 编码，0 主手 / 1 副手）
// 包名与字段格式以 getPacketNames() 和实测为准
buf.writeVarInt(0)
buf.sendPacket("HandSwingC2SPacket")
```

- `sendPacket(...)`：把缓冲区内容按指定包类型构造成包**发给服务器**（serverbound 包）。
- `receivePacket(...)`：把缓冲区内容构造成包后**让客户端当作从服务器收到的包来处理**
  （clientbound 包，适合本地模拟测试；具体行为以实测为准）。

```javascript
// 本地伪造一条收包，测试脚本对它的反应（不会经过服务器）
const buf2 = Client.createPacketByteBuffer()
// ...按该包格式写入字段...
buf2.receivePacket("某个S2C包名")
```

!!! warning "字段格式必须完全正确"
    `sendPacket` / `receivePacket` / `toPacket` 都是按缓冲区当前内容反序列化出包对象，
    字段顺序、类型、个数有任何偏差，轻则抛异常，重则把错误数据发给服务器导致被踢/被封。

## PacketByteBufferHelper 方法总表

以下按用途分组，全部对照 d.ts 38159–39165 行核实（均 1.8.4 加入，特别标注除外）。
写方法基本都返回自身，可链式调用。

### 获取与转换

| 成员 | 说明 |
| --- | --- |
| `Client.createPacketByteBuffer()` | 最常用的获取方式：空缓冲区 |
| `event.getPacketBuffer()` | 从 `RecvPacket` / `SendPacket` 事件获取，内容为该包的序列化数据 |
| `new PacketByteBufferHelper()` / `(buffer)` / `(packet)` | 直接构造：空的 / 包装已有缓冲区 / 从包创建 |
| 静态 `getPacketName(packet)` | 反查包对象的名字 |
| 静态 `BUFFER_TO_PACKET` 字段、`init()`、`main(args)` | 内部实现用，官方注释："别碰"（Don't touch this here!） |
| `toPacket()` | 按创建时的包类型重建包；helper 不是从包创建的则返回 `null` |
| `toPacket(packetName)` / `toPacket(clazz)` | 按指定包名/类重建包 |
| `getPacketNames()` | 全部包名列表 |
| `getPacketId(packetClass)` | 包的 ID |
| `getNetworkStateId(packetClass)` | 包所属网络阶段的 ID |
| `isClientbound(packetClass)` / `isServerbound(packetClass)` | 判断包的方向（服务器→客户端 / 客户端→服务器） |

### 发送与接收

| 方法 | 说明 |
| --- | --- |
| `sendPacket()` | 按创建时的包类型，把缓冲区构造成包发给服务器 |
| `sendPacket(packetName)` / `sendPacket(clazz)` | 按指定包名/类发送 |
| `receivePacket()` | 按创建时的包类型，让客户端接收缓冲区构造的包 |
| `receivePacket(packetName)` / `receivePacket(clazz)` | 按指定包名/类接收 |

### 基础类型读写

顺序读写会移动指针；`setXxx` / `getXxx` 按索引直接存取，不动指针。

| 类型 | 顺序写 | 顺序读 | 按索引写 | 按索引读 |
| --- | --- | --- | --- | --- |
| boolean | `writeBoolean(v)` | `readBoolean()` | `setBoolean(i, v)` | `getBoolean(i)` |
| char | `writeChar(v)` | `readChar()` | `setChar(i, v)` | `getChar(i)` |
| byte | `writeByte(v)` | `readByte()` / `readUnsignedByte()` | `setByte(i, v)` | `getByte(i)` / `getUnsignedByte(i)` |
| short | `writeShort(v)` | `readShort()` / `readUnsignedShort()` | `setShort(i, v)` | `getShort(i)` / `getUnsignedShort(i)` |
| medium（3 字节） | `writeMedium(v)` | `readMedium()` / `readUnsignedMedium()` | `setMedium(i, v)` | `getMedium(i)` / `getUnsignedMedium(i)` |
| int | `writeInt(v)` | `readInt()` / `readUnsignedInt()` | `setInt(i, v)` | `getInt(i)` / `getUnsignedInt(i)` |
| long | `writeLong(v)` | `readLong()` | `setLong(i, v)` | `getLong(i)` |
| float | `writeFloat(v)` | `readFloat()` | `setFloat(i, v)` | `getFloat(i)` |
| double | `writeDouble(v)` | `readDouble()` | `setDouble(i, v)` | `getDouble(i)` |

原始字节操作：

| 方法 | 说明 |
| --- | --- |
| `writeBytes(bytes)` / `setBytes(i, bytes)` | 写入字节数组（顺序 / 按索引） |
| `readBytes(length)` / `getBytes(i, length)` | 读出指定长度的字节数组（顺序 / 按索引） |
| `writeZero(length)` / `setZero(i, length)` | 写入指定数量的 0 |
| `skipBytes(length)` | 读指针前进 length 字节 |

### 变长与 Minecraft 常用类型

| 写 | 读 | 说明 |
| --- | --- | --- |
| `writeVarInt(i)` | `readVarInt()` | 变长 int，MC 协议里最常见的整数编码 |
| `writeVarLong(l)` | `readVarLong()` | 变长 long |
| `writeString(s)` / `writeString(s, maxLength)` | `readString()` / `readString(maxLength)` | 字符串；超长抛异常 |
| `writeIdentifier(id)` | `readIdentifier()` | 标识符，如 `"minecraft:stone"` |
| `writeUuid(uuid)` | `readUuid()` | UUID（写入参数是字符串） |
| `writeNbt(nbtCompound)` | `readNbt()` | NBT 复合标签 |
| `writeEnumConstant(constant)` | `readEnumConstant(enumClass)` | Java 枚举常量 |
| `writeDate(date)` | `readDate()` | `java.util.Date` |
| `writeInstant(instant)` | `readInstant()` | `java.time.Instant` |
| `writePublicKey(key)` | `readPublicKey()` | 公钥 |
| `writeBitSet(bitSet)` | `readBitSet()` | `java.util.BitSet` |
| `writeRegistryKey(key)` | `readRegistryKey(registry)` | 注册表键 |

### 位置与世界类型

| 写 | 读 | 说明 |
| --- | --- | --- |
| `writeBlockPos(pos)` / `writeBlockPos(x, y, z)` | `readBlockPos()` | 方块坐标，读出 `BlockPosHelper` |
| `writeChunkPos(x, z)` / `writeChunkPos(chunk)` | `readChunkPos()` / `readChunkHelper()` | 区块坐标；前者读出 `[x, z]` 数组，后者读出 `ChunkHelper`（可能为 `null`） |
| `writeGlobalPos(dimension, pos)` / `writeGlobalPos(dimension, x, y, z)` | —— | 带维度的全局坐标（`overworld` / `the_nether` / `the_end`），d.ts 未声明对应读取方法 |
| `writeBlockHitResult(hitResultHelper)`（1.9.1） / `writeBlockHitResult(pos, direction, blockPos, missed, insideBlock)` | `readBlockHitResultHelper()`（1.9.1） | 方块命中结果（交互包常用） |
| `writeBlockHitResult(原始对象)` | `readBlockHitResult()` / `readBlockHitResultMap()` | 已弃用，官方建议改用上一行的 1.9.1 版本 |

### 集合与可空值

带 `writer` / `reader` 参数的方法需要传 `JavaWrapper.methodToJava(...)`，
回调收到底层缓冲区对象，由你决定每个元素怎么读写。

| 写 | 读 | 说明 |
| --- | --- | --- |
| `writeCollection(collection, writer)` | `readList(reader)` | 任意集合 / 读出 List |
| `writeIntList(list)` | `readIntList()` | 整数列表 |
| `writeMap(map, keyWriter, valueWriter)` | `readMap(keyReader, valueReader)` | Map |
| —— | `forEachInCollection(reader)` | 逐个读取集合元素，对每个元素调用 reader，返回自身 |
| `writeOptional(value, writer)` | `readOptional(reader)` | `java.util.Optional` 包装 |
| `writeNullable(value, writer)` | `readNullable(reader)` | 可空值，读出值或 `null` |
| `writeByteArray(bytes)` | `readByteArray()` / `readByteArray(maxSize)` | 字节数组；超过 maxSize 抛异常 |
| `writeIntArray(ints)` | `readIntArray()` / `readIntArray(maxSize)` | int 数组；带 maxSize 的重载在 d.ts 里声明返回自身 |
| `writeLongArray(longs)` | `readLongArray()` / `readLongArray(maxSize)` | long 数组 |

### 指针（索引）控制

| 方法 | 说明 |
| --- | --- |
| `readerIndex()` / `setReaderIndex(i)` | 读指针位置 / 设置读指针 |
| `writerIndex()` / `setWriterIndex(i)` | 写指针位置 / 设置写指针 |
| `setIndices(readerIndex, writerIndex)` | 同时设置两个指针 |
| `markReaderIndex()` / `resetReaderIndex()` | 给读指针做书签 / 回到书签 |
| `markWriterIndex()` / `resetWriterIndex()` | 给写指针做书签 / 回到书签 |
| `resetIndices()` | 两个指针都回到各自上次标记的位置 |
| `clear()` | 两个指针归零；数据不清空，新写入会覆盖旧数据 |
| `reset()` | 恢复到 helper 创建时的状态 |

## 常见用途

- **调试插件服交互**：很多插件服的自定义界面、商店、抽奖靠自定义 payload 和界面包驱动，
  监听 `RecvPacket` 并打印 `type` 可以搞清楚"点了按钮之后服务器到底发了什么"。
- **抓取界面数据**：容器/交易界面的内容本质上都来自服务器发的包，在包层面能拿到
  界面渲染之前的原始数据。
- **本地模拟测试**：用 `receivePacket` 伪造收包，不连服务器也能测脚本对某类包的反应。
- **过滤刷屏内容**：取消特定的粒子、音效等装饰性收包（注意上面的状态不同步警告）。

!!! tip "先抓包再动手"
    想操作某个包，第一步永远是：开一个只打印 `type` 的监听（配合事件过滤器或者
    在回调里去重），在游戏里做一次目标操作，看触发了哪些包，再去读它的字段。
