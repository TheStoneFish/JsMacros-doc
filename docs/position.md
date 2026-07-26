---
icon: lucide/axis-3d
---

# 位置与向量

JsMacros 里所有"坐标"相关的东西都绕不开这五个类：`Pos2D`、`Pos3D`（点）、`Vec2D`、`Vec3D`（向量）和 `BlockPosHelper`（整数方块坐标）。本页按 `JsMacros-2.1.0.d.ts` 逐一核对了它们的全部方法，并给出实战几何套路：算距离、判断"在不在我面前"、算朝向某点的 yaw/pitch、HUD 方位显示。

## 三种类型，各管一摊

| 类型 | 是什么 | 坐标精度 | 典型来源 |
| --- | --- | --- | --- |
| `Pos2D` | 二维点 (x, y) | 小数 | `entity.getChunkPos()`、屏幕坐标 |
| `Pos3D` | 三维点 (x, y, z) | 小数 | `entity.getPos()`、`entity.getEyePos()` |
| `Vec2D` | 二维向量 = 起点 + 终点 | 小数 | 两个 `Pos2D` 相连 |
| `Vec3D` | 三维向量 = 起点 + 终点 | 小数 | `PositionCommon.createLookingVector(...)` |
| `BlockPosHelper` | 方块坐标 (x, y, z) | 整数 | `entity.getBlockPos()`、[方块](blocks.md)相关 API |

三者的关系一句话：**点表示"在哪"，向量表示"从哪到哪"（方向 + 长度），方块坐标是取整后的点**。

JsMacros 的向量与很多数学库不同——它同时存着**起点和终点**两组坐标，方向分量是"终点 − 起点"（即 `getDeltaX/Y/Z()`）。模长、点积、叉积、yaw/pitch 都基于这个差值计算。

### 互转速查

| 想要的转换 | 写法 |
| --- | --- |
| `Pos2D` → `Pos3D` | `pos2.to3D()`（z 补 0） |
| `Pos3D` → `Vec3D` | `pos.toVector(start)` / `pos.toReverseVector(end)` |
| `Vec3D` → 两个 `Pos3D` | `vec.getStart()` / `vec.getEnd()` |
| `Vec2D` → `Vec3D` | `vec2.to3D()` |
| `BlockPosHelper` → `Pos3D` | `blockPos.toPos3D()` |
| `Pos3D` → `BlockPosHelper` | 无直接方法，用 `PositionCommon.createBlockPos(Math.floor(p.x), Math.floor(p.y), Math.floor(p.z))` |

!!! note "这些对象从哪来"
    - 手动创建：本页下面的 `PositionCommon` 入口函数。
    - [实体](entities.md)：`entity.getPos()` / `getEyePos()` 返回 `Pos3D`，`getBlockPos()` 返回 `BlockPosHelper`，`getChunkPos()` 返回 `Pos2D`（注意：`Pos2D` 只有 x、y 两个字段，所以区块的 z 坐标存在 `y` 里）。
    - [方块](blocks.md)：`World.getBlock(...)` 等返回的 `BlockHelper` / 扫描结果里到处是 `BlockPosHelper`。

## PositionCommon：入口函数

创建位置和向量的函数都在全局库 **`PositionCommon`** 下。

!!! warning "不是 Player.createPos"
    旧版教程常写 `Player.createPos(...)`。在 2.1.0 的类型定义中，这些函数属于 `PositionCommon`，`Player` 下没有它们（[玩家](player.md)页也有说明）。

| 函数 | 返回 | 说明 | 版本 |
| --- | --- | --- | --- |
| `createPos(x, y, z)` | `Pos3D` | 三维点 | 1.6.3 |
| `createPos(x, y)` | `Pos2D` | 二维点 | 1.6.3 |
| `createVec(x1, y1, z1, x2, y2, z2)` | `Vec3D` | 从点 1 指向点 2 的三维向量 | 1.6.3 |
| `createVec(x1, y1, x2, y2)` | `Vec2D` | 从点 1 指向点 2 的二维向量 | 1.6.3 |
| `createLookingVector(entity)` | `Vec3D` | 实体视线方向的单位向量 | 1.8.4 |
| `createLookingVector(yaw, pitch)` | `Vec3D` | 由 yaw/pitch 算出的方向单位向量 | 1.8.4 |
| `createBlockPos(x, y, z)` | `BlockPosHelper` | 整数方块坐标（参数是 `int`） | 1.8.4 |

```javascript
const p3 = PositionCommon.createPos(100.5, 64, -20.5)      // Pos3D
const p2 = PositionCommon.createPos(10, 20)                // Pos2D
const vec = PositionCommon.createVec(0, 64, 0, 10, 70, 10) // Vec3D：从 (0,64,0) 指向 (10,70,10)
const block = PositionCommon.createBlockPos(100, 64, -21)  // BlockPosHelper
const look = PositionCommon.createLookingVector(Player.getPlayer()) // 玩家正在看的方向
```

## Pos2D / Pos3D（点）

`Pos3D` 继承自 `Pos2D`，所以 `Pos2D` 的方法它全有（三维重载覆盖了大部分）。所有运算都**返回新对象**，不修改原对象，可以放心链式调用。

### 字段与常量

| 成员 | 类型 | 说明 |
| --- | --- | --- |
| `x` / `y`（`Pos3D` 另有 `z`） | `number` | 坐标，可直接读：`pos.x` |
| `Pos2D.ZERO` / `Pos3D.ZERO` | 静态常量 | 原点 |
| `getX()` / `getY()` / `getZ()` | `number` | 等价于读字段 |

### Pos2D 方法

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `add(pos)` / `add(x, y)` | `Pos2D` | 逐坐标相加 |
| `sub(pos)` / `sub(x, y)` | `Pos2D` | 逐坐标相减 |
| `multiply(pos)` / `multiply(x, y)` | `Pos2D` | 逐坐标相乘 |
| `divide(pos)` / `divide(x, y)` | `Pos2D` | 逐坐标相除 |
| `scale(scale)` | `Pos2D` | 所有坐标乘同一个数 |
| `to3D()` | `Pos3D` | 升维，z 补 0 |
| `toVector()` | `Vec2D` | 原点 → 本点 的向量 |
| `toVector(start_pos)` / `toVector(start_x, start_y)` | `Vec2D` | 给定起点 → 本点 |
| `toReverseVector()` | `Vec2D` | 本点 → 原点 |
| `toReverseVector(end_pos)` / `toReverseVector(end_x, end_y)` | `Vec2D` | 本点 → 给定终点 |
| `compareTo(o)` | `number` | 排序比较用 |

### Pos3D 新增/覆盖的方法

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `add(pos3)` / `add(x, y, z)` | `Pos3D` | 三维相加 |
| `add(pos2)` / `add(x, y)` | `Pos2D` | :warning: 两参数重载会**降维**成 `Pos2D`（丢掉 z） |
| `sub(pos3)` / `sub(x, y, z)` | `Pos3D` | 三维相减（两参数版同样降维） |
| `multiply(...)` / `divide(...)` | `Pos3D` / `Pos2D` | 同上，三参三维、两参降维 |
| `scale(scale)` | `Pos3D` | 整体缩放 |
| `toVector()` | `Vec3D` | 原点 → 本点 |
| `toVector(start_pos3)` / `toVector(start_pos2)` / `toVector(x, y, z)` | `Vec3D` | 给定起点 → 本点（起点是 `Pos2D` 时 z 按 0 算） |
| `toVector(start_x, start_y)` | `Vec2D` | 两参数版降维成 `Vec2D` |
| `toReverseVector()` / `toReverseVector(end...)` | `Vec3D` / `Vec2D` | 本点 → 终点，重载规律同 `toVector` |
| `compareTo(o)` | `number` | 排序比较用 |

以下方法是给引擎内部或 Java 互操作用的，日常脚本用不到：静态 `fromVec3d(vec3d, converter)`、`toRawBlockPos(converter)`、`toMojangDoubleVector(converter)`（两者的无参版本已废弃）、`convert(converter)`。

!!! warning "Pos2D / Pos3D 没有 distanceTo"
    和一些旧文档写的不同，2.1.0 的 `Pos2D` / `Pos3D` **没有** `distanceTo` 方法。算两点距离用向量模长：

    ```javascript
    const a = PositionCommon.createPos(0, 64, 0)
    const b = PositionCommon.createPos(10, 70, 10)
    const dist = a.toReverseVector(b).getMagnitude()  // a → b 的向量长度
    ```

    `BlockPosHelper` 和实体的 `EntityHelper` 倒是有 `distanceTo`，见下文。

```javascript
const pos = PositionCommon.createPos(1, 64, 2)
const above = pos.add(0, 1, 0)        // Pos3D (1, 65, 2)
const doubled = pos.scale(2)          // Pos3D (2, 128, 4)
const diff = above.sub(pos)           // Pos3D (0, 1, 0)
Chat.log(`x=${above.x}, y=${above.getY()}, z=${above.z}`)
```

## Vec2D / Vec3D（向量）

`Vec3D` 继承自 `Vec2D`。构造时给"起点 + 终点"两组坐标；方向分量 = 终点 − 起点。

### 字段与构造

| 成员 | 说明 |
| --- | --- |
| `x1, y1 (, z1)` | 起点坐标字段 |
| `x2, y2 (, z2)` | 终点坐标字段 |
| `new Vec2D(x1, y1, x2, y2)` / `new Vec2D(start, end)` | 也可用 `PositionCommon.createVec(...)` |
| `new Vec3D(x1, y1, z1, x2, y2, z2)` / `new Vec3D(start, end)` | 同上 |

### Vec2D 方法

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getX1()` / `getY1()` / `getX2()` / `getY2()` | `number` | 起点/终点坐标 |
| `getDeltaX()` / `getDeltaY()` | `number` | 方向分量（终点 − 起点） |
| `getStart()` / `getEnd()` | `Pos2D` | 起点/终点转成点 |
| `getMagnitude()` | `number` | 模长（向量长度） |
| `getMagnitudeSq()` | `number` | 模长的平方（只比大小时用它，省一次开方） |
| `add(vec)` / `add(x1, y1, x2, y2)` | `Vec2D` | 起点、终点坐标分别相加 |
| `multiply(vec)` / `multiply(x1, y1, x2, y2)` | `Vec2D` | 逐坐标相乘 |
| `scale(scale)` | `Vec2D` | 整体缩放（模长也乘 scale） |
| `dotProduct(vec)` | `number` | 点积（基于方向分量） |
| `reverse()` | `Vec2D` | 反向 |
| `normalize()` | `Vec2D` | 方向不变、模长归一化为 1 |
| `to3D()` | `Vec3D` | 升维，z 补 0 |
| `compareTo(other)` | `number` | 排序比较用 |

### Vec3D 新增/覆盖的方法

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getZ1()` / `getZ2()` / `getDeltaZ()` | `number` | z 方向的起点/终点/分量 |
| `getStart()` / `getEnd()` | `Pos3D` | 起点/终点 |
| `add(vec3)` / `add(x1, y1, z1, x2, y2, z2)` | `Vec3D` | 三维相加 |
| `add(vec2)` / `add(x1, y1, x2, y2)` | `Vec2D` | :warning: 二维重载降维 |
| `addStart(pos)` / `addStart(x, y, z)` | `Vec3D` | 只平移起点 |
| `addEnd(pos)` / `addEnd(x, y, z)` | `Vec3D` | 只平移终点（= 只改方向/长度） |
| `multiply(...)` | `Vec3D` / `Vec2D` | 重载规律同 `add` |
| `scale(scale)` | `Vec3D` | 整体缩放 |
| `normalize()` | `Vec3D` | 归一化为单位向量 |
| `getYaw()` | `number` | 这个方向对应的 MC 水平角 yaw（度） |
| `getPitch()` | `number` | 这个方向对应的 MC 俯仰角 pitch（度） |
| `dotProduct(vec3)` / `dotProduct(vec2)` | `number` | 点积 |
| `crossProduct(vec3)` | `Vec3D` | 叉积（得到同时垂直于两向量的向量） |
| `reverse()` | `Vec3D` | 反向 |
| `toMojangFloatVector()` | `org.joml.Vector3f` | 转原版 JOML 向量，与原版/渲染代码互操作用 |
| `compareTo(o)` | `number` | 排序比较用 |

!!! note "getYaw() / getPitch() 的废弃标记"
    d.ts 中无参的 `getYaw()` / `getPitch()` 被标记为废弃，推荐的新签名 `getYaw(mathHelper)` / `getPitch(mathHelper)` 需要一个 `IMathHelper` 实例——但脚本环境里并没有现成的获取入口，所以**脚本里继续用无参版本即可**。如果哪天它被移除，可以手算（MC 角度约定见下文"实用几何"）：

    ```javascript
    const dx = vec.getDeltaX(), dy = vec.getDeltaY(), dz = vec.getDeltaZ()
    const yaw = Math.atan2(-dx, dz) * 180 / Math.PI
    const pitch = -Math.atan2(dy, Math.sqrt(dx * dx + dz * dz)) * 180 / Math.PI
    ```

!!! tip "向量没有 sub / divide"
    `Vec2D` / `Vec3D` 只有 `add` / `multiply` / `scale`。要"减去一个向量"就 `a.add(b.reverse())`；要除以标量就 `v.scale(1 / n)`。

```javascript
const player = Player.getPlayer()
const look = PositionCommon.createLookingVector(player)  // 单位向量
Chat.log(`视线方向: Δ(${look.getDeltaX().toFixed(2)}, ${look.getDeltaY().toFixed(2)}, ${look.getDeltaZ().toFixed(2)})`)
Chat.log(`yaw=${look.getYaw().toFixed(1)}, pitch=${look.getPitch().toFixed(1)}`)

// 视线前方 5 格的点：眼睛位置 + 方向分量 × 5
const eye = player.getEyePos()
const ahead = eye.add(look.getDeltaX() * 5, look.getDeltaY() * 5, look.getDeltaZ() * 5)
Chat.log(`前方 5 格: (${ahead.x.toFixed(1)}, ${ahead.y.toFixed(1)}, ${ahead.z.toFixed(1)})`)
```

## BlockPosHelper：整数方块坐标

完整方法表在[方块](blocks.md)页，这里只列与 `Pos3D` 互转及距离相关的部分：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `toPos3D()` | `Pos3D` | 转小数点坐标 |
| `distanceTo(entity)` | `number` | 到实体的距离 |
| `distanceTo(pos)` | `number` | 到另一 `BlockPosHelper` 或 `Pos3D` 的距离 |
| `distanceTo(x, y, z)` | `number` | 到给定坐标的距离 |
| `up()` / `down()` / `north()` / `south()` / `east()` / `west()`（可带距离参数） | `BlockPosHelper` | 相邻方块 |
| `offset(direction)` / `offset(direction, distance)` / `offset(x, y, z)` | `BlockPosHelper` | 按方向或位移偏移 |
| `toNetherCoords()` / `toOverworldCoords()` | `BlockPosHelper` | 下界/主世界坐标换算 |

```javascript
const feet = Player.getPlayer().getBlockPos()   // BlockPosHelper
const below = feet.down()                       // 脚下方块
const p3 = feet.toPos3D()                       // 转 Pos3D 继续做小数运算
Chat.log(`到 (0,64,0) 的距离: ${feet.distanceTo(0, 64, 0).toFixed(1)}`)

// Pos3D → BlockPosHelper：手动取整
const pos = Player.getPlayer().getPos()
const bp = PositionCommon.createBlockPos(Math.floor(pos.x), Math.floor(pos.y), Math.floor(pos.z))
```

## 方向字符串（Direction）

不少 API 的方向参数是字符串，d.ts 里定义为：

```ts
type Direction = "up" | "down" | "north" | "south" | "east" | "west"
```

例如 `BlockPosHelper.offset(direction)`，以及方块交互（点击方块的哪个面）：

```javascript
const pos = Player.getPlayer().getBlockPos().down()
Player.getInteractionManager().interactBlock(pos.getX(), pos.getY(), pos.getZ(), "up", false)
```

## 实用几何

先记住 MC 的角度约定，后面全都用得上：

- **yaw（水平角）**：`0` = 南（+z），`90` = 西（−x），`±180` = 北（−z），`-90` = 东（+x）。数值增大 = 向右转。
- **pitch（俯仰角）**：`-90` = 正上方，`0` = 水平，`90` = 正下方。

### 两个实体 / 两点之间的距离

```javascript
const player = Player.getPlayer()

for (const e of World.getEntities()) {
    if (e.getUUID() === player.getUUID()) continue
    // EntityHelper 自带 distanceTo（还有 (pos3d)、(blockPos)、(x,y,z) 重载）
    const d = player.distanceTo(e)
    if (d < 10) {
        Chat.log(`${e.getName().getString()} 距离 ${d.toFixed(1)} 格`)
    }
}

// 纯坐标版本：两点连成向量取模长
const a = PositionCommon.createPos(0, 64, 0)
const b = player.getPos()
Chat.log(`到 (0,64,0) 的距离: ${a.toReverseVector(b).getMagnitude().toFixed(1)}`)
```

!!! tip "只比较远近时用 getMagnitudeSq()"
    比较"谁更近"不需要真实距离，用模长平方可以省掉开方运算：`vecA.getMagnitudeSq() < vecB.getMagnitudeSq()`。

### 判断实体是否在玩家面前（视野锥）

原理：视线方向单位向量与"玩家眼睛 → 实体"单位向量做**点积**，点积 = 夹角的余弦。点积大于 `cos(半角)` 就说明实体在这个锥形视野内。

```javascript
/**
 * 实体是否在玩家面前 maxAngle 度的视野锥内
 */
function isInFront(entity, maxAngle) {
    const player = Player.getPlayer()
    const look = PositionCommon.createLookingVector(player).normalize()
    const toEntity = player.getEyePos().toReverseVector(entity.getEyePos()).normalize()
    const cos = look.dotProduct(toEntity)
    return cos > Math.cos(maxAngle * Math.PI / 180)
}

for (const e of World.getEntities()) {
    if (e.getUUID() === Player.getPlayer().getUUID()) continue
    if (isInFront(e, 30) && Player.getPlayer().distanceTo(e) < 20) {
        Chat.log(`§a准心附近: ${e.getName().getString()}`)
    }
}
```

### 朝向某个坐标需要的 yaw / pitch

把"眼睛 → 目标"连成 `Vec3D`，`getYaw()` / `getPitch()` 直接给出角度，配合 [`player.lookAt`](player.md) 转头：

```javascript
const player = Player.getPlayer()
const target = PositionCommon.createPos(100, 70, -50)

const vec = player.getEyePos().toReverseVector(target)  // 眼睛 → 目标
const yaw = vec.getYaw()
const pitch = vec.getPitch()
Chat.log(`看向目标需要 yaw=${yaw.toFixed(1)}, pitch=${pitch.toFixed(1)}`)

player.lookAt(yaw, pitch)
// 其实 lookAt 还有更直接的重载：player.lookAt(100, 70, -50)
// 以及方向字符串版：player.lookAt("north")
```

### 视线方向向量 createLookingVector

`createLookingVector` 是"角度 → 方向向量"的逆操作，返回长度为 1 的 `Vec3D`：

```javascript
const player = Player.getPlayer()

// 玩家（或任意实体）当前视线
const look = PositionCommon.createLookingVector(player)

// 任意 yaw/pitch 的方向，比如"正北偏上 45 度"
const dir = PositionCommon.createLookingVector(180, -45)

// 常见用法：沿视线取一个点（例如放 Draw3D 标记、判断落点）
const eye = player.getEyePos()
const point = eye.add(look.getDeltaX() * 3, look.getDeltaY() * 3, look.getDeltaZ() * 3)
Chat.log(`视线前方 3 格: (${point.x.toFixed(1)}, ${point.y.toFixed(1)}, ${point.z.toFixed(1)})`)
```

## 完整示例：HUD 显示到目标点的距离与方位

把目标点的距离和相对方位（前/左/右/后 + 上/下）常驻显示在屏幕上。渲染 API 的细节见 [HUD 渲染](hud.md)。

```javascript
// ===== 配置：目标坐标 =====
const TARGET = PositionCommon.createPos(0, 64, 0)

// 把任意角度差规范到 -180 ~ 180
function wrapDeg(a) {
    return ((a % 360) + 540) % 360 - 180
}

// 相对角度 → 八方位文字（正数在右，负数在左）
function dirText(rel) {
    const table = ["前", "右前", "右", "右后", "后", "左后", "左", "左前"]
    const idx = Math.round((((rel % 360) + 360) % 360) / 45) % 8
    return table[idx]
}

const d2d = Hud.createDraw2D()
let distLine, dirLine

// 元素必须在 setOnInit 里创建（窗口尺寸变化会重建，详见 hud.md 常见坑）
d2d.setOnInit(JavaWrapper.methodToJava(() => {
    distLine = d2d.addText("目标距离: ...", 8, 8, 0xFFFFFF, true)
    dirLine = d2d.addText("方位: ...", 8, 20, 0x55FF55, true)
}))
d2d.register()

JsMacros.on("Tick", JavaWrapper.methodToJava(() => {
    const p = Player.getPlayer()
    if (!p || !distLine) return

    const vec = p.getEyePos().toReverseVector(TARGET)  // 眼睛 → 目标
    const dist = vec.getMagnitude()
    const rel = wrapDeg(vec.getYaw() - p.getYaw())     // 目标方位相对玩家朝向的偏角

    // 垂直方向提示
    const dy = vec.getDeltaY()
    const vertical = dy > 2 ? " ↑上方" : dy < -2 ? " ↓下方" : ""

    distLine.setText(`目标距离: ${dist.toFixed(1)} 格`)
    dirLine.setText(`方位: ${dirText(rel)} (${rel.toFixed(0)}°)${vertical}`)
}))

// 做成服务时加上这行自动收尾（见 services.md）；
// 临时测试的话，跑完手动执行 Hud.clearDraw2Ds() 清理
// event.unregisterOnStop(true, d2d)
```

!!! tip "想在世界里直接画出目标？"
    用 `Hud.createDraw3D()` 在目标点画方框/连线更直观，见 [HUD 渲染](hud.md)的 Draw3D 部分。

## 相关页面

- [玩家](player.md)——`Player.getPlayer()`、`lookAt`、射线检测
- [实体](entities.md)——`getPos()` / `getEyePos()` / `distanceTo` 等实体侧方法
- [方块](blocks.md)——`BlockPosHelper` 完整方法表与世界方块操作
- [HUD 渲染](hud.md)——Draw2D / Draw3D 的完整用法
