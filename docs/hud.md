---
icon: lucide/panel-top
---

# HUD 渲染（Draw2D 与 Draw3D）

`Hud` 全局对象负责一切"画在屏幕上/画在世界里"的事情：状态栏、坐标显示、物品图标、ESP 方框、实体连线、世界内悬浮面板等。本页覆盖 `Hud` 命名空间、`Draw2D`、`Draw3D` 及全部 2D / 3D 元素类与 Builder。

!!! note "本页与《脚本屏幕》的分工"
    `Hud.createScreen()` 创建的可交互 GUI（按钮、输入框）请看[脚本屏幕](screen.md)。本页只讲**渲染**：不可交互的覆盖层和世界内绘制。

## 核心概念

| 对象 | 画在哪里 | 典型用途 |
| --- | --- | --- |
| `Draw2D` | 屏幕覆盖层（HUD），随 GUI 缩放 | 状态栏、坐标/FPS 显示、物品图标、小地图边框 |
| `Draw3D` | 世界空间，跟随摄像机 | 方块高亮（ESP）、路径线、实体追踪线、世界内面板 |
| `Surface` | 世界中的一块 2D 画布（属于 Draw3D） | 悬浮告示牌、头顶血条、全息面板 |

三条规则贯穿全页：

1. **创建 ≠ 显示**。`Hud.createDraw2D()` / `Hud.createDraw3D()` 只是创建对象，必须调用对象自己的 `register()` 才开始渲染，`unregister()` 停止渲染。
2. **add 方法返回元素对象**。保存返回值，之后可以随时 `setText` / `setPos` / `setColor`，不需要删了重加。
3. **脚本停止不会自动清理**。普通脚本结束后，已注册的 Draw2D/Draw3D 仍会留在屏幕上；建议做成[服务](services.md)并用 `event.unregisterOnStop()` 收尾（见下文）。

### 生命周期与清理

`Draw2D` 和 `Draw3D` 都实现了 `Registrable` 接口，服务事件的 `unregisterOnStop` 可以直接接收它们：

```javascript
// 服务脚本里（event 是 Service 事件）
const d2d = Hud.createDraw2D()
d2d.register()

// 服务停止时：清掉本脚本注册的事件监听 + 注销 d2d
event.unregisterOnStop(true, d2d)
```

等价的手写形式：

```javascript
event.stopListener = JavaWrapper.methodToJava(() => d2d.unregister())
```

!!! tip "执行顺序"
    d.ts 注释注明：停止服务时的顺序是 `stopListener` → 关闭事件监听 → 注销 Registrable → `postStopListener`。只要调用过 `unregisterOnStop`，服务就不会在脚本跑到末尾时自动退出——正适合常驻 HUD。

旧版全局注册函数 `Hud.registerDraw2D()` / `Hud.unregisterDraw2D()` / `Hud.registerDraw3D()` / `Hud.unregisterDraw3D()` 自 1.6.5 起已废弃，新代码一律用对象自己的 `register()` / `unregister()`。

## 颜色写法：0xRRGGBB 与 0xAARRGGBB

所有颜色参数都是一个 int，用十六进制字面量最直观：

```javascript
0xFF0000            // 红色（RRGGBB）
0x80FF0000          // 半透明红（AARRGGBB，AA=0x80 即 128/255 透明度）
```

- **带独立 `alpha` 参数的重载**：`color` 写 `0xRRGGBB`，`alpha` 传 `0`（全透明）到 `255`（不透明）。
- **不带 `alpha` 参数、但注释标明"含 alpha 通道"的重载**（如 5 参数版 `addRect`、`addLine`）：必须写 `0xAARRGGBB`，否则 alpha 位是 0，**画出来是全透明的，看不见**。
- 文字颜色一般直接写 `0xRRGGBB` 即可。
- 很多 Builder 还支持 `color(r, g, b)` / `color(r, g, b, a)`，每个分量 0–255。

!!! warning "矩形画不出来？八成是 alpha"
    `d2d.addRect(0, 0, 100, 20, 0xFF0000)` 是隐形的——5 参数版需要 `0xFFFF0000`。要么补上 AA 位，要么改用 6 参数版 `addRect(0, 0, 100, 20, 0xFF0000, 255)`。

## 快速上手

### 示例一：坐标 + FPS 信息栏（推荐做成服务）

```javascript
const d2d = Hud.createDraw2D()

let posLine, fpsLine

// 窗口尺寸变化时 Draw2D 会清空并重新初始化，
// 所以元素创建要放在 setOnInit 里（详见"常见坑"）
d2d.setOnInit(JavaWrapper.methodToJava(() => {
    d2d.addRect(4, 4, 150, 32, 0x000000, 120)          // 半透明底板
    posLine = d2d.addText("加载中...", 8, 8, 0xFFFFFF, true)
    fpsLine = d2d.addText("FPS ...", 8, 20, 0x55FF55, true)
}))
d2d.register()

JsMacros.on("Tick", JavaWrapper.methodToJava(() => {
    const p = Player.getPlayer()
    if (!p || !posLine) return
    posLine.setText(`XYZ ${p.getX().toFixed(1)} / ${p.getY().toFixed(1)} / ${p.getZ().toFixed(1)}`)
    fpsLine.setText(Client.getFPS())   // 返回 MC 的 fps 调试字符串
}))

// 服务脚本收尾；如果只是临时脚本测试，手动跑 d2d.unregister() 清理
event.unregisterOnStop(true, d2d)
```

### 示例二：高亮方块（ESP 框）

```javascript
const d3d = Hud.createDraw3D()

// 方式 A：Builder 链式，forBlock 直接框住整格方块
d3d.boxBuilder()
    .forBlock(100, 64, 100)
    .color(0x00FF00, 255)      // 轮廓：不透明绿
    .fillColor(0x00FF00, 60)   // 填充：半透明绿
    .fill(true)
    .cull(false)               // false = 隔墙也能看见
    .buildAndAdd()

// 方式 B：直接 addBox（color, alpha, fillColor, fillAlpha, fill）
d3d.addBox(102, 64, 100, 103, 65, 101, 0xFF0000, 255, 0xFF0000, 40, true)

d3d.register()
event.unregisterOnStop(true, d3d)
```

### 示例三：给附近苦力怕拉追踪线

```javascript
const d3d = Hud.createDraw3D().register()

const entities = World.getEntities(30)   // 30 格内的实体
if (entities) {
    for (const e of entities) {
        if (e.getType() === "minecraft:creeper") {
            d3d.addEntityTraceLine(e, 0x00FF00, 180)
        }
    }
}

event.unregisterOnStop(true, d3d)
```

## Hud 命名空间速查

| 函数 | 返回 | 说明 |
| --- | --- | --- |
| `createDraw2D()` | `Draw2D` | 新建 2D 覆盖层 |
| `createDraw3D()` | `Draw3D` | 新建 3D 绘制对象 |
| `listDraw2Ds()` | `JavaList<IDraw2D>` | 当前已注册的所有 Draw2D |
| `listDraw3Ds()` | `JavaList<Draw3D>` | 当前已注册的所有 Draw3D |
| `clearDraw2Ds()` | `void` | 清空 Draw2D 渲染列表（包括别的脚本注册的） |
| `clearDraw3Ds()` | `void` | 清空 Draw3D 渲染列表 |
| `registerDraw2D(overlay)` | `void` | :warning: 已废弃，用 `Draw2D.register()` |
| `unregisterDraw2D(overlay)` | `void` | :warning: 已废弃，用 `Draw2D.unregister()` |
| `registerDraw3D(draw)` | `void` | :warning: 已废弃，用 `Draw3D.register()` |
| `unregisterDraw3D(draw)` | `void` | :warning: 已废弃，用 `Draw3D.unregister()` |
| `createScreen(title, dirtBG)` | `ScriptScreen` | 创建脚本屏幕，见[脚本屏幕](screen.md) |
| `openScreen(s)` | `void` | 打开屏幕，传 `null` 关闭当前屏幕 |
| `getOpenScreen()` | `IScreen \| null` | 当前打开的屏幕对象 |
| `getOpenScreenName()` | `string \| null` | 当前屏幕名称 |
| `isContainer()` | `boolean` | 当前是否容器界面 |
| `createTexture(width, height, name)` | `CustomImage` | 创建空白画布纹理 |
| `createTexture(path, name)` | `CustomImage` | 从图片文件加载纹理（`path` 为绝对路径） |
| `getRegisteredTextures()` | `JavaMap<string, CustomImage>` | 所有已注册的自定义纹理（只读） |
| `getScaleFactor()` | `number` | 当前 GUI 缩放倍数 |
| `getMouseX()` / `getMouseY()` | `number` | 鼠标坐标（缩放后坐标系） |
| `getWindowWidth()` / `getWindowHeight()` | `number` | 窗口物理像素尺寸 |

### 自定义纹理

```javascript
// 从文件加载（d.ts 标注 path 为图片文件的绝对路径）
const image = Hud.createTexture("E:/图片/icon.png", "my_icon")

// 或创建一张 64x64 的空白画布（CustomImage 上有画点画线等方法）
const canvas = Hud.createTexture(64, 64, "my_canvas")

Chat.log(Hud.getRegisteredTextures().keySet())
```

创建后有两种用法：

```javascript
const d2d = Hud.createDraw2D().register()

// 用法 1：imageBuilder().fromCustomImage() 自动填好全部纹理参数（推荐）
d2d.imageBuilder().fromCustomImage(image).pos(10, 40).size(32, 32).buildAndAdd()

// 用法 2：addImage 手动引用纹理 ID
d2d.addImage(10, 80, 32, 32, image.getIdentifier(), 0, 0, 64, 64, 64, 64)
```

### 屏幕和鼠标信息

```javascript
Chat.log(Hud.getOpenScreenName())
Chat.log(`鼠标: ${Hud.getMouseX()}, ${Hud.getMouseY()}`)
Chat.log(`窗口: ${Hud.getWindowWidth()}x${Hud.getWindowHeight()}  缩放: ${Hud.getScaleFactor()}`)
```

## Draw2D 完整参考

### 创建、注册与初始化

```javascript
const d2d = Hud.createDraw2D()
d2d.setOnInit(JavaWrapper.methodToJava((self) => {
    // self 就是这个 Draw2D；在这里创建元素
    self.addText("Hello JsMacros", 10, 10, 0xFFFFFF, true)
}))
d2d.setOnFailInit(JavaWrapper.methodToJava((err) => Chat.log(`HUD 初始化失败: ${err}`)))
d2d.register()
```

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `register()` | `this` | 开始渲染（会触发一次 init） |
| `unregister()` | `this` | 停止渲染 |
| `setOnInit(onInit)` | `Draw2D` | 初始化回调；**窗口尺寸变化或注册时调用，调用前会清空全部元素** |
| `setOnFailInit(catchInit)` | `Draw2D` | init 回调抛异常时收到错误信息字符串 |
| `init()` | `void` | 手动触发初始化（清空并重建） |
| `setVisible(visible)` | `this` | 显示/隐藏整个覆盖层（不注销） |
| `isVisible()` | `boolean` | 是否可见 |
| `setZIndex(zIndex)` / `getZIndex()` | `void` / `number` | 多个 Draw2D 之间的层级 |
| `getWidth()` / `getHeight()` | `number` | **缩放后**的屏幕尺寸（见"屏幕缩放坐标系"） |

!!! warning "元素创建放进 setOnInit"
    Draw2D 在窗口大小改变时会**清空所有元素并重新调用 onInit**。如果你在脚本主体里直接 `addText`，玩家一拖窗口/改 GUI 缩放，元素就全没了。正确姿势：元素创建写在 `setOnInit` 回调里，并在回调里把引用存到外部变量。

### 元素管理

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getTexts()` / `getRects()` / `getLines()` / `getItems()` / `getImages()` | `JavaList<...>` | 按类型列出元素 |
| `getDraw2Ds()` | `JavaList<Draw2DElement>` | 嵌套的子 Draw2D |
| `getElements()` | `JavaList<RenderElement>` | 所有元素（只读副本） |
| `getElementsByZIndex()` | `Iterator<RenderElement>` | 按 z-index 顺序迭代 |
| `removeElement(e)` | `Draw2D` | 移除任意类型元素 |
| `reAddElement(e)` | 元素本身 | 把 `removeElement` 移除的元素加回来 |
| `removeText(t)` / `removeRect(r)` / `removeLine(l)` / `removeItem(i)` / `removeImage(i)` / `removeDraw2D(d)` | `Draw2D` | 按类型移除 |

所有元素都实现 `RenderElement` 接口（`getZIndex()`）；z-index 大的画在上面。

---

### 文本 Text

**直接添加**——`text` 可以是字符串，也可以是 [TextHelper](chat.md)（保留 §格式/JSON 样式）：

```javascript
addText(text, x, y, color, shadow)                                  // 基础版
addText(text, x, y, color, zIndex, shadow)                          // + 层级
addText(text, x, y, color, shadow, scale, rotation)                 // + 缩放旋转
addText(text, x, y, color, zIndex, shadow, scale, rotation)         // 全参数
```

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `text` | `string` \| `TextHelper` | 内容 |
| `x`, `y` | `int` | 左上角坐标（缩放后坐标系） |
| `color` | `int` | `0xRRGGBB` |
| `zIndex` | `int` | 层级，大者在上 |
| `shadow` | `boolean` | 是否带阴影 |
| `scale` | `double` | 缩放倍数（1 为原大） |
| `rotation` | `double` | 顺时针旋转角度 |

!!! warning "zIndex 和 shadow 的位置"
    带 `zIndex` 的重载是 `addText(text, x, y, color, zIndex, shadow)`——`zIndex` 插在 `color` 和 `shadow` **中间**。把 `true` 写到第 5 个参数会被当成 `zIndex=1`，别写反。

```javascript
const d2d = Hud.createDraw2D().register()
const line = d2d.addText("血量", 10, 10, 0xFF5555, true)

JsMacros.on("Tick", JavaWrapper.methodToJava(() => {
    const p = Player.getPlayer()
    if (p) line.setText(`HP ${p.getHealth().toFixed(0)}`)
}))
```

**Builder 链式写法**——`textBuilder()` / `textBuilder(text)`：

```javascript
d2d.textBuilder("警告！")
    .pos(10, 30)
    .color(255, 85, 85)      // 也可 .color(0xFF5555)
    .shadow(true)
    .scale(1.5)
    .zIndex(10)
    .buildAndAdd()           // build() 只创建不添加
```

**Text 元素方法速查**（`set` 系列均可运行时改）：

| 方法 | 说明 |
| --- | --- |
| `setText(string)` / `setText(TextHelper)` / `getText()` | 改/读内容（`getText` 返回 `TextHelper`） |
| `setPos(x, y)` / `setX(x)` / `setY(y)` / `getX()` / `getY()` | 位置 |
| `setColor(color)` / `getColor()` | 颜色 |
| `setShadow(bool)` / `hasShadow()` | 阴影 |
| `setScale(scale)` / `getScale()` | 缩放（scale ≤ 0 抛异常） |
| `setRotation(deg)` / `getRotation()` | 旋转 |
| `setRotateCenter(bool)` / `isRotatingCenter()` | 是否绕中心旋转 |
| `getWidth()` / `getHeight()` | 文本渲染尺寸（做右对齐/居中很有用） |
| `setZIndex(z)` / `getZIndex()` | 层级 |

**Text$Builder 方法速查**：`text(string | TextHelper | TextBuilder)`、`pos(x, y)`、`x(x)`、`y(y)`、`color(color)`、`color(r,g,b)`、`color(r,g,b,a)`、`scale(s)`、`rotation(deg)`、`rotateCenter(bool)`、`shadow(bool)`、`zIndex(z)`，以及对应 `getXxx()` 读取器和 `getWidth()`/`getHeight()`；最后 `build()` 或 `buildAndAdd()`。

---

### 矩形 Rect

**直接添加**——`(x1, y1)`、`(x2, y2)` 是两个对角：

```javascript
addRect(x1, y1, x2, y2, color)                        // color 必须含 alpha：0xAARRGGBB
addRect(x1, y1, x2, y2, color, alpha)                 // color 0xRRGGBB + alpha 0-255
addRect(x1, y1, x2, y2, color, alpha, rotation)
addRect(x1, y1, x2, y2, color, alpha, rotation, zIndex)
```

```javascript
d2d.addRect(8, 8, 160, 40, 0x000000, 120)             // 半透明黑底板
d2d.addRect(0, 0, 50, 50, 0x80FF0000)                 // 5 参数版：AARRGGBB
```

**Builder**——`rectBuilder()` / `rectBuilder(x, y, width, height)`（注意 Builder 用宽高，`width(w)` 等价于 `x2 = x1 + w`）：

```javascript
d2d.rectBuilder(10, 10, 100, 20)
    .color(0, 0, 0, 140)
    .zIndex(-1)
    .buildAndAdd()
```

**Rect 元素方法速查**：`setPos(x1,y1,x2,y2)`、`setPos1(x1,y1)`、`setPos2(x2,y2)`、`setX1/setY1/setX2/setY2` 与 getter、`setWidth(w)`/`setHeight(h)`/`setSize(w,h)`、`setColor(color)`、`setColor(color, alpha)`、`setAlpha(a)`、`getColor()`/`getAlpha()`、`setRotation(deg)`、`setRotateCenter(bool)`、`setZIndex(z)`。

**Rect$Builder 方法速查**：`pos1(x1,y1)`、`pos2(x2,y2)`、`pos(x1,y1,x2,y2)`、`x1/y1/x2/y2`、`width/height/size`、`color(color)`、`color(color,alpha)`、`color(r,g,b)`、`color(r,g,b,a)`、`alpha(a)`、`rotation(deg)`、`rotateCenter(bool)`、`zIndex(z)` + 各 getter。

---

### 直线 Line（2D）

```javascript
addLine(x1, y1, x2, y2, color)                        // color 可含 alpha
addLine(x1, y1, x2, y2, color, zIndex)
addLine(x1, y1, x2, y2, color, width)                 // width 为 double 线宽
addLine(x1, y1, x2, y2, color, zIndex, width)
addLine(x1, y1, x2, y2, color, width, rotation)
addLine(x1, y1, x2, y2, color, zIndex, width, rotation)
```

!!! note "重载区分"
    第 6 个参数既可能是 `zIndex`（int）也可能是 `width`（double）。传 `2` 会被当作 zIndex，传 `2.0`／`2.5` 才是线宽。拿不准就用 `lineBuilder()`。

```javascript
d2d.addLine(0, 50, 200, 50, 0xFFFFFFFF, 2.0)          // 白色 2px 横线
```

**Builder**——`lineBuilder()` / `lineBuilder(x1, y1, x2, y2)`：

```javascript
d2d.lineBuilder(10, 60, 110, 60)
    .color(0x55FFFF)
    .width(3.0)
    .buildAndAdd()
```

**Line 元素方法速查**：`setPos(x1,y1,x2,y2)`、`setPos1/setPos2`、`setX1/setY1/setX2/setY2`、`setColor(color)`、`setColor(color,alpha)`、`setAlpha(a)`、`setWidth(double)`、`setRotation(deg)`、`setRotateCenter(bool)`、`setZIndex(z)` + 各 getter。

**Line$Builder 方法速查**：`pos1/pos2/pos`、`x1/y1/x2/y2`、`width(double)`、`color(color)`、`color(color,alpha)`、`color(r,g,b)`、`color(r,g,b,a)`、`alpha(a)`、`rotation(deg)`、`rotateCenter(bool)`、`zIndex(z)`。

---

### 物品 Item

`id` 是物品 ID（`minecraft:` 前缀可省略），或直接传背包里的 `ItemStackHelper`：

```javascript
addItem(x, y, id)
addItem(x, y, zIndex, id)
addItem(x, y, id, overlay)                            // overlay: 显示数量/耐久条
addItem(x, y, zIndex, id, overlay)
addItem(x, y, id, overlay, scale, rotation)
addItem(x, y, zIndex, id, overlay, scale, rotation)
// 以上 6 个重载把 id 换成 ItemStackHelper 同样成立，共 12 个
```

```javascript
d2d.addItem(10, 100, "diamond_sword")

// 显示主手物品（含数量/耐久 overlay）
const held = Player.getPlayer().getMainHand()
d2d.addItem(10, 120, held, true)
```

**Builder**——`itemBuilder()` / `itemBuilder(item)`：

```javascript
d2d.itemBuilder()
    .item("minecraft:golden_apple", 64)   // item(id, count) 可设堆叠数
    .pos(10, 140)
    .overlayVisible(true)
    .overlayText("x64")                   // 自定义角标文字（会自动开启 overlay）
    .scale(1.5)
    .buildAndAdd()
```

**Item 元素方法速查**：`setItem(ItemStackHelper)`、`setItem(id, count)`、`getItem()`、`setPos(x,y)`、`setX/setY`、`setScale(s)`（非法值抛异常）、`setRotation(deg)`、`setRotateCenter(bool)`、`setOverlay(bool)`/`shouldShowOverlay()`、`setOverlayText(str)`/`getOverlayText()`、`setZIndex(z)`。

**Item$Builder 方法速查**：`pos/x/y`、`item(ItemStackHelper)`、`item(id)`、`item(id, count)`、`getItem()`、`overlayText(str)`、`overlayVisible(bool)`、`scale(s)`、`rotation(deg)`、`rotateCenter(bool)`、`zIndex(z)`。

---

### 图片 Image

`addImage` 参数多但规律固定：屏幕上的位置尺寸 + 纹理里的采样区域。`id` 是纹理 ID，格式如资源包路径：`assets/minecraft/textures/gui/recipe_book.png` → `minecraft:textures/gui/recipe_book.png`；自定义纹理用 `CustomImage.getIdentifier()`。

```javascript
addImage(x, y, width, height, id, imageX, imageY, regionWidth, regionHeight, textureWidth, textureHeight)
addImage(x, y, width, height, zIndex, id, imageX, imageY, regionWidth, regionHeight, textureWidth, textureHeight)
addImage(x, y, width, height, id, imageX, imageY, regionWidth, regionHeight, textureWidth, textureHeight, rotation)
addImage(x, y, width, height, zIndex, id, imageX, imageY, regionWidth, regionHeight, textureWidth, textureHeight, rotation)
addImage(x, y, width, height, zIndex, color, id, imageX, imageY, regionWidth, regionHeight, textureWidth, textureHeight, rotation)
addImage(x, y, width, height, zIndex, alpha, color, id, imageX, imageY, regionWidth, regionHeight, textureWidth, textureHeight, rotation)
```

| 参数 | 说明 |
| --- | --- |
| `x`, `y`, `width`, `height` | 画到屏幕上的位置和尺寸（左上角） |
| `id` | 纹理 ID |
| `imageX`, `imageY` | 采样区域在纹理里的左上角 |
| `regionWidth`, `regionHeight` | 采样区域大小 |
| `textureWidth`, `textureHeight` | 整张纹理的尺寸 |
| `rotation` | 顺时针角度 |
| `color` / `alpha` | 染色与透明度（0xRRGGBB + 0–255） |

```javascript
// 画原版图标纹理集的左上 16x16（整张 icons.png 是 256x256）
d2d.addImage(10, 160, 32, 32, "minecraft:textures/gui/icons.png", 0, 0, 16, 16, 256, 256)
```

**Builder**——`imageBuilder()` / `imageBuilder(id)`，自定义图片首选 `fromCustomImage`：

```javascript
const tex = Hud.createTexture("E:/图片/logo.png", "logo")
d2d.imageBuilder()
    .fromCustomImage(tex)        // 自动填 identifier/region/textureSize
    .pos(10, 200)
    .size(48, 48)
    .buildAndAdd()
```

**Image 元素方法速查**：`setImage(id, imageX, imageY, regionW, regionH, texW, texH)`、`getImage()`、`setPos(x,y)`、`setPos(x,y,w,h)`、`setX/setY`、`setWidth/setHeight/setSize`、`setColor(color)`、`setColor(color, alpha)`、`getColor()/getAlpha()`、`setRotation(deg)`、`setRotateCenter(bool)`、`setZIndex(z)`。

**Image$Builder 方法速查**：`fromCustomImage(img)`、`identifier(id)`、`pos/x/y`、`size/width/height`、`imagePos(imageX, imageY)`/`imageX`/`imageY`、`regionSize(w,h)`/`regionWidth`/`regionHeight`、`regions(x,y,w,h)`、`regions(x,y,w,h,texW,texH)`、`textureSize(w,h)`/`textureWidth`/`textureHeight`、`color(color)`、`color(color,alpha)`、`color(r,g,b)`、`color(r,g,b,a)`、`alpha(a)`、`rotation(deg)`、`rotateCenter(bool)`、`zIndex(z)`。

---

### 嵌套画布 Draw2DElement

把一个 Draw2D 塞进另一个 Draw2D（或屏幕）里，做成可整体移动/缩放的"组件"：

```javascript
addDraw2D(draw2D, x, y, width, height)          // 返回 Draw2DElement 包装
addDraw2D(draw2D, x, y, width, height, zIndex)
removeDraw2D(draw2DElement)
```

```javascript
const panel = Hud.createDraw2D()
panel.setOnInit(JavaWrapper.methodToJava(() => {
    panel.addRect(0, 0, 80, 30, 0x000000, 150)
    panel.addText("面板", 4, 4, 0xFFFFFF, true)
}))

const root = Hud.createDraw2D().register()
const child = root.addDraw2D(panel, 20, 20, 80, 30)
child.setScale(1.5)     // 整体放大
```

存在循环嵌套（A 含 B、B 又含 A）时添加会失败。Builder 版本：`draw2DBuilder(draw2D)`，方法有 `pos/x/y`、`size/width/height`、`scale(s)`、`rotation(deg)`、`rotateCenter(bool)`、`zIndex(z)`。

**Draw2DElement 方法速查**：`getDraw2D()`、`setPos(x,y)`、`setX/setY`、`setSize(w,h)`/`setWidth/setHeight`、`setScale(s)`、`setRotation(deg)`、`setRotateCenter(bool)`、`setZIndex(z)` + 各 getter。

---

### 世界坐标投影 WorldPosWrapper（2.1.0 新增）

把任意 2D 元素"钉"到一个世界坐标上：元素仍画在屏幕层，但位置每帧按世界坐标投影计算——适合做穿墙可见的文字标签（3D 的 Surface 做不到穿墙显示文字）。

```javascript
addWorldPosWrapped(pos, base)          // pos: Pos3D
addWorldPosWrapped(x, y, z, base)      // base: 任意 RenderElement（Text/Rect/Item/...）
```

```javascript
const d2d = Hud.createDraw2D().register()

// 注意用 build() 只创建、不直接加入 d2d，由包装器负责渲染
const label = d2d.textBuilder("目标点").color(0xFFD700).shadow(true).build()
const wrapped = d2d.addWorldPosWrapped(100.5, 65.5, 100.5, label)

// 也可以让它跟随实体（此时 pos 变成相对实体的偏移）
// wrapped.setFollowed(someEntity)
```

**WorldPosWrapper 方法/字段速查**：`setPos(pos)` / `setPos(x, y, z)`、`setBase(element)` 换被包装的元素、`setFollowed(entity)` 跟随实体（传 `null` 取消）、`setScaleThreshold(threshold)` 距离缩放阈值、字段 `shouldRemove` 置 `true` 可让它自行移除；静态字段 `WorldPosWrapper.globalScaleThreshold` 修改全局默认阈值（只影响之后创建的包装器）。

---

### 元素对齐 Alignable

所有 2D 元素和它们的 Builder 都实现了 `Alignable`，可以按父容器或其他元素对齐，省去手算坐标：

| 方法 | 说明 |
| --- | --- |
| `alignHorizontally(alignment)` / `alignHorizontally(alignment, offset)` | 相对父容器水平对齐：`"left"` / `"center"` / `"right"` / `"35%"` |
| `alignVertically(alignment)` / `alignVertically(alignment, offset)` | 相对父容器垂直对齐：`"top"` / `"center"` / `"bottom"` / 百分比 |
| `alignHorizontally(other, alignment[, offset])` | 相对另一个元素对齐，格式 `"左侧On右侧"`，如 `"LeftOnCenter"` |
| `alignVertically(other, alignment[, offset])` | 同上，如 `"BottomOnTop"`（把自己放到对方上面） |
| `align(h, v)` / `align(h, hOffset, v, vOffset)` | 水平 + 垂直一次搞定 |
| `align(other, h, v)` / `align(other, h, hOffset, v, vOffset)` | 相对其他元素 |
| `moveTo(x, y)` / `moveToX(x)` / `moveToY(y)` | 直接挪位置 |
| `getScaledWidth()` / `getScaledHeight()` | 元素缩放后的尺寸 |
| `getScaledLeft()` / `getScaledTop()` / `getScaledRight()` / `getScaledBottom()` | 缩放后四边位置 |
| `getParentWidth()` / `getParentHeight()` | 父容器尺寸 |

对齐字符串大小写不敏感；`Alignable.parsePercentage("35%")` 可解析百分比（非法返回 -1）。

```javascript
// 右上角显示，距边缘 4px（offset 正值向右/向下）
const fps = d2d.textBuilder("FPS").color(0x55FF55).shadow(true)
    .alignHorizontally("right", -4)
    .alignVertically("top", 4)
    .buildAndAdd()

// 把一行字精确放到某个矩形下方
const box = d2d.rectBuilder(10, 10, 80, 20).color(0, 0, 0, 140).buildAndAdd()
d2d.textBuilder("说明文字").alignVertically(box, "TopOnBottom", 2)
    .alignHorizontally(box, "CenterOnCenter").buildAndAdd()
```

## Draw3D 完整参考

### 创建、注册与元素管理

```javascript
const d3d = Hud.createDraw3D()
d3d.register()      // 开始渲染
d3d.unregister()    // 停止渲染
```

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `register()` / `unregister()` | `this` | 开关渲染 |
| `getBoxes()` | `JavaList<Box>` | 所有方框 |
| `getLines()` | `JavaList<Line3D>` | 所有 3D 线 |
| `getTraceLines()` | `JavaList<TraceLine>` | 所有追踪线 |
| `getEntityTraceLines()` | `JavaList<EntityTraceLine>` | 所有实体追踪线 |
| `getDraw2Ds()` | `JavaList<Surface>` | 所有 Surface |
| `clear()` | `void` | 清空全部元素 |
| `reAddElement(element)` | `void` | 重新加入被移除的元素 |
| `addBox(box)` / `addLine(line)` / `addTraceLine(line)` / `addSurface(surface)` | `void` | 加入预先 `build()` 好的元素对象 |
| `removeBox(b)` / `removeLine(l)` / `removeTraceLine(t)` / `removeDraw2D(surface)` | — | 移除元素 |
| `boxBuilder()` / `boxBuilder(blockPos)` / `boxBuilder(x, y, z)` | `Box$Builder` | 方框 Builder |
| `lineBuilder()` | `Line3D$Builder` | 3D 线 Builder |
| `traceLineBuilder()` | `TraceLine$Builder` | 追踪线 Builder |
| `entityTraceLineBuilder()` | `EntityTraceLine$Builder` | 实体追踪线 Builder |
| `surfaceBuilder()` | `Surface$Builder` | Surface Builder |
| `addEntityFollower(base, entity)` | `EntityFollowWrapper` | 让 3D 元素跟随实体（2.1.0） |

!!! note "cull 参数"
    多个 3D 元素都有 `cull`（剔除）参数：`cull: true` 表示被方块挡住的部分不画（正常深度测试）；`cull: false` 表示**穿墙可见**。默认多为 `false`。

---

### 方框 Box 与 addPoint

**直接添加**——两组对角坐标（double，可以带小数）：

```javascript
addBox(x1, y1, z1, x2, y2, z2, color, fillColor, fill)
addBox(x1, y1, z1, x2, y2, z2, color, fillColor, fill, cull)
addBox(x1, y1, z1, x2, y2, z2, color, alpha, fillColor, fillAlpha, fill)
addBox(x1, y1, z1, x2, y2, z2, color, alpha, fillColor, fillAlpha, fill, cull)
```

| 参数 | 说明 |
| --- | --- |
| `color` / `alpha` | 轮廓线颜色 / 透明度 0–255 |
| `fillColor` / `fillAlpha` | 填充面颜色 / 透明度 |
| `fill` | 是否画填充面（`false` 只画线框） |
| `cull` | 是否被地形遮挡 |

```javascript
// 半透明绿色整格框
d3d.addBox(100, 64, 100, 101, 65, 101, 0x00FF00, 255, 0x00FF00, 40, true)
```

**addPoint**——以某点为中心画一个小立方体（`边长 = 2 * radius`），标记坐标点很方便：

```javascript
addPoint(pos, radius, color)                       // pos: Pos3D
addPoint(x, y, z, radius, color)
addPoint(x, y, z, radius, color, alpha, cull)
```

```javascript
d3d.addPoint(100.5, 65.5, 100.5, 0.1, 0xFF00FF)
```

**Builder**——`boxBuilder()`，`forBlock` 是高亮整格方块的捷径：

```javascript
d3d.boxBuilder()
    .forBlock(120, 70, -30)          // 等价 pos1/pos2 框住这一格
    .color(0xFFAA00, 255)
    .fillColor(0xFFAA00, 50)
    .fill(true)
    .cull(false)
    .buildAndAdd()
```

**Box 元素方法速查**：`setPos(x1,y1,z1,x2,y2,z2)`、`setPosToBlock(blockPos)`、`setPosToBlock(x,y,z)`、`setPosToPoint(pos, radius)`、`setPosToPoint(x,y,z,radius)`、`setColor(color)`、`setColor(color,alpha)`、`setAlpha(a)`、`setFillColor(c)`、`setFillColor(c,alpha)`、`setFillAlpha(a)`、`setFill(bool)`；字段 `pos`、`color`、`fillColor`、`fill`、`cull`。

**Box$Builder 方法速查**：`pos1(Pos3D | BlockPosHelper | x,y,z)`、`pos2(同)`、`pos(x1..z2)`、`pos(pos1, pos2)`、`forBlock(x,y,z)`、`forBlock(blockPos)`、`color(color)`、`color(color,alpha)`、`color(r,g,b)`、`color(r,g,b,a)`、`alpha(a)`、`fillColor(...)` 同样 4 种重载、`fillAlpha(a)`、`fill(bool)`、`cull(bool)`、`build()`、`buildAndAdd()`。

---

### 直线 Line3D

```javascript
addLine(x1, y1, z1, x2, y2, z2, color)
addLine(x1, y1, z1, x2, y2, z2, color, cull)
addLine(x1, y1, z1, x2, y2, z2, color, alpha)
addLine(x1, y1, z1, x2, y2, z2, color, alpha, cull)
removeLine(l)
```

```javascript
// 从脚下到目标点画一条红线
const p = Player.getPlayer()
d3d.addLine(p.getX(), p.getY(), p.getZ(), 100.5, 65, 100.5, 0xFF0000, 255)
```

**Builder**——`lineBuilder()`：

```javascript
d3d.lineBuilder()
    .pos1(Player.getPlayer().getPos())
    .pos2(100, 65, 100)
    .color(255, 0, 0, 255)
    .cull(false)
    .buildAndAdd()
```

**Line3D 元素方法速查**：`setPos(x1,y1,z1,x2,y2,z2)`、`setColor(color)`、`setColor(color,alpha)`、`setAlpha(a)`；字段 `pos`、`color`、`cull`。

**Line3D$Builder 方法速查**：`pos1(Pos3D | BlockPosHelper | x,y,z)`、`pos2(同)`、`pos(x1..z2)`、`pos(pos1,pos2)`、`color` 4 种重载、`alpha(a)`、`cull(bool)`、`build()`、`buildAndAdd()`。

---

### 追踪线 TraceLine

从玩家准星（视角中心）拉一条线到目标坐标，找地点神器：

```javascript
addTraceLine(x, y, z, color)
addTraceLine(x, y, z, color, alpha)
addTraceLine(pos, color)              // pos: Pos3D
addTraceLine(pos, color, alpha)
removeTraceLine(traceLine)
```

```javascript
const trace = d3d.addTraceLine(100.5, 65.5, 100.5, 0x00FFFF, 200)
// 之后可随时改目标：
trace.setPos(200.5, 70.5, -50.5)
```

**Builder**——`traceLineBuilder()`：`pos(Pos3D | BlockPosHelper | x,y,z)`、`color` 4 种重载、`alpha(a)`、`build()`、`buildAndAdd()`。

**TraceLine 元素方法速查**：`setPos(x,y,z)`、`setPos(pos)`、`setColor(color)`、`setColor(color,alpha)`、`setAlpha(a)`。

---

### 实体追踪线 EntityTraceLine

`TraceLine` 的子类，目标换成实体并自动跟随；实体消失后线会自动移除：

```javascript
addEntityTraceLine(entity, color)
addEntityTraceLine(entity, color, alpha)
addEntityTraceLine(entity, color, alpha, yOffset)     // yOffset: 线端点在实体身上的高度偏移
```

```javascript
const entities = World.getEntities(50)
if (entities) {
    for (const e of entities) {
        if (e.getType() === "minecraft:villager") {
            d3d.addEntityTraceLine(e, 0x00FF00, 160, 1.0)   // 指向村民腰部往上 1 格
        }
    }
}
```

**EntityTraceLine 元素方法速查**：继承 TraceLine 的全部方法，另有 `setEntity(entity)`（可传 `null`）、`setYOffset(offset)`、`shouldRemove()`（实体没了返回 `true`）。

**EntityTraceLine$Builder 方法速查**：`entity(entity)`、`yOffset(offset)`、`color` 4 种重载、`alpha(a)`、`build()` / `build(entity)`、`buildAndAdd()` / `buildAndAdd(entity)`。

---

### 实体跟随 EntityFollowWrapper（2.1.0 新增）

把任意 3D 元素包装成"跟着实体走"（元素坐标按相对实体位置的偏移理解）：

```javascript
const target = Player.getPlayer()   // 演示：跟随自己；实际可用任意 EntityHelper

// build() 创建但不加入，元素坐标写成相对偏移
const ring = d3d.boxBuilder()
    .pos(-0.5, 0, -0.5, 0.5, 0.1, 0.5)
    .color(0xFF5555, 255)
    .fill(false)
    .build()

const follower = d3d.addEntityFollower(ring, target)
```

**方法/字段速查**：`setBase(base)` 换元素、`setEntity(entity)` 换目标、`shouldRemove()`；字段 `base`、`entity`、`shouldRemove`。

## Surface：世界中的 2D 画布

`Surface` 继承自 `Draw2D`——上面讲的 **所有 2D 元素方法（addText/addRect/addItem/textBuilder/……）在 Surface 上全部可用**，只是这块画布被放到了世界里。

**通过 Draw3D 创建**（`x, y, z` 是画布左上角；`width/height` 单位是方块；`xRot/yRot/zRot` 是三轴旋转角度）：

```javascript
addDraw2D(x, y, z)
addDraw2D(x, y, z, width, height)
addDraw2D(x, y, z, xRot, yRot, zRot)
addDraw2D(x, y, z, xRot, yRot, zRot, width, height)
addDraw2D(x, y, z, xRot, yRot, zRot, width, height, minSubdivisions)
addDraw2D(x, y, z, xRot, yRot, zRot, width, height, minSubdivisions, renderBack)
addDraw2D(x, y, z, xRot, yRot, zRot, width, height, minSubdivisions, renderBack, cull)
removeDraw2D(surface)
```

```javascript
const d3d = Hud.createDraw3D()

const surface = d3d.surfaceBuilder()
    .pos(100, 66, 100)
    .size(2, 1)                 // 2 格宽 1 格高
    .rotateToPlayer(true)       // 始终面向玩家（广告牌效果）
    .renderBack(true)           // 背面也渲染
    .cull(false)                // 穿墙可见
    .minSubdivisions(64)        // 提高内部画布分辨率
    .buildAndAdd()

// Surface 就是一个 Draw2D：内部像素尺寸用 getWidth()/getHeight() 取
const w = surface.getWidth(), h = surface.getHeight()
surface.addRect(0, 0, w, h, 0x000000, 140)
surface.addText("欢迎回家", 2, 2, 0xFFD700, false)

d3d.register()
event.unregisterOnStop(true, d3d)
```

!!! tip "内部分辨率"
    画布的"内部像素"由尺寸和 `minSubdivisions`（最小细分数）共同决定，`getWidth()` / `getHeight()` 返回实际可用像素。文字太糊就调大 `minSubdivisions`。

**Surface 专有方法/字段速查**（其余全部继承 Draw2D）：

| 成员 | 说明 |
| --- | --- |
| `setPos(pos: Pos3D)` / `setPos(blockPos)` / `setPos(x, y, z)` | 移动画布 |
| `setRotations(x, y, z)` | 三轴旋转角度 |
| `setRotateToPlayer(bool)` / `doesRotateToPlayer()` | 始终面向玩家 |
| `setRotateCenter(bool)` / `isRotatingCenter()` | 是否绕中心旋转 |
| `setSizes(x, y)` / `getSizes()` | 尺寸（方块单位，`Pos2D`） |
| `setMinSubdivisions(n)` / `getMinSubdivisions()` | 最小细分数 |
| `bindToEntity(entity)` / `getBoundEntity()` | 绑定实体（跟随移动，传 `null` 解绑） |
| `setBoundOffset(pos)` / `setBoundOffset(x, y, z)` / `getBoundOffset()` | 相对绑定实体的偏移 |
| `zIndexScale` | 字段；元素 zIndex 换算成世界偏移的比例，默认 `1/1000`，元素之间闪面（z-fighting）就调大 |
| `renderBack` / `cull` | 字段；背面渲染 / 深度剔除 |
| 只读字段 `pos`、`rotations` | 当前位置与旋转 |

**Surface$Builder 方法速查**：`pos(Pos3D | BlockPosHelper | x,y,z)`、`bindToEntity(entity)`、`boundOffset(pos)` / `boundOffset(x,y,z)`、`xRotation/yRotation/zRotation/rotation(x,y,z)`、`rotateCenter(bool)`、`rotateToPlayer(bool)`、`width(w)`/`height(h)`/`size(w,h)`（double，方块单位）、`minSubdivisions(n)`、`renderBack(bool)`、`cull(bool)`、`zIndex(zIndexScale)`、`build()`、`buildAndAdd()` + 各 getter。

**实体头顶悬浮牌示例**：

```javascript
const d3d = Hud.createDraw3D()
const p = Player.getPlayer()

const tag = d3d.surfaceBuilder()
    .bindToEntity(p)
    .boundOffset(0, 2.3, 0)      // 头顶上方 0.3 格
    .size(1.5, 0.4)
    .rotateToPlayer(true)
    .renderBack(true)
    .minSubdivisions(48)
    .buildAndAdd()

tag.addText("我是标签", 2, 2, 0xFFFFFF, true)
d3d.register()
event.unregisterOnStop(true, d3d)
```

## 屏幕缩放坐标系

Draw2D 使用的是 **GUI 缩放后的坐标系**，和原版 HUD（血条、快捷栏）一致：

- `Hud.getWindowWidth()` / `getWindowHeight()`：窗口的**物理像素**尺寸。
- `Hud.getScaleFactor()`：视频设置里的 GUI 缩放倍数（"自动"会根据窗口大小取 1–4 等）。
- `d2d.getWidth()` / `d2d.getHeight()`：**缩放后**尺寸，即 `窗口像素 ÷ 缩放倍数`。你给 `addText`/`addRect` 传的坐标都在这个坐标系里。

```javascript
const d2d = Hud.createDraw2D().register()
Chat.log(`物理: ${Hud.getWindowWidth()}x${Hud.getWindowHeight()}`)
Chat.log(`缩放: x${Hud.getScaleFactor()}`)
Chat.log(`Draw2D 坐标系: ${d2d.getWidth()}x${d2d.getHeight()}`)
```

这意味着：

1. 同一个 `(x, y)` 在不同 GUI 缩放下的**屏幕相对位置不同**。想贴边、居中，别写死坐标，在 `setOnInit` 里用 `d2d.getWidth()`/`getHeight()` 计算，或直接用上文"元素对齐 Alignable"一节的 `align("right", "bottom")`。
2. 玩家改缩放/拖窗口时会触发 Draw2D 重新 init（清空元素 + 调 onInit），这正是把创建逻辑放进 `setOnInit` 的原因——回调里拿到的 `getWidth()` 永远是最新值。

```javascript
// 一个永远贴在右下角的水印
const d2d = Hud.createDraw2D()
d2d.setOnInit(JavaWrapper.methodToJava(() => {
    const text = d2d.addText("MyHUD v1.0", 0, 0, 0xAAAAAA, false)
    text.setPos(d2d.getWidth() - text.getWidth() - 2, d2d.getHeight() - 10)
}))
d2d.register()
```

## 常见坑

!!! warning "Tick 里反复 add，元素越堆越多"
    最常见的错误：在 `Tick` 监听里每帧 `addText`，几秒后屏幕糊满重影、帧率暴跌。**元素只创建一次，之后用引用 `setText`/`setPos` 更新**：

    ```javascript
    // 错误 ✗
    JsMacros.on("Tick", JavaWrapper.methodToJava(() => {
        d2d.addText(`X: ${Player.getPlayer().getX()}`, 10, 10, 0xFFFFFF, true)
    }))

    // 正确 ✓
    const line = d2d.addText("", 10, 10, 0xFFFFFF, true)
    JsMacros.on("Tick", JavaWrapper.methodToJava(() => {
        line.setText(`X: ${Player.getPlayer().getX().toFixed(1)}`)
    }))
    ```

!!! warning "忘记 unregister，HUD 永远留在屏幕上"
    脚本结束不等于渲染结束。测试时残留了一堆？`Hud.clearDraw2Ds()` + `Hud.clearDraw3Ds()` 一键清场（注意会把其他脚本的也清掉）。正式脚本请做成服务并 `event.unregisterOnStop(true, d2d, d3d)`。

!!! warning "窗口一变，元素全没了"
    Draw2D 在窗口尺寸变化时清空并重跑 onInit。直接在脚本主体 add 的元素会消失且不会回来。元素创建必须放进 `setOnInit`。

!!! warning "zIndex 与 shadow / width 的参数位置"
    很多 2D 方法的 `zIndex` 重载都是把它插在中间：`addText(text, x, y, color, zIndex, shadow)`、`addLine(x1, y1, x2, y2, color, zIndex, width)`。参数一多容易错位，拿不准就换 Builder 写法，链式方法名不会歧义。

!!! warning "颜色看不见 = alpha 是 0"
    5 参数 `addRect`、`addLine` 的 color 必须是 `0xAARRGGBB`。`0xFF0000` 的 AA 位为 0，完全透明。

!!! tip "性能建议"
    - 大量 3D 元素（比如矿物 ESP 的几百个 Box）放进**同一个** Draw3D，不要每个方块 new 一个 Draw3D。
    - 批量增删时可先 `unregister()`，改完再 `register()`。
    - 不再用的元素及时 `removeElement` / `removeBox`，或者整个 `clear()`。

## 相关页面

- [脚本屏幕](screen.md)——可交互 GUI（按钮、输入框、滑条）
- [服务](services.md)——常驻脚本与 `unregisterOnStop`
- [聊天与文本](chat.md)——`TextHelper` / `TextBuilder`，配合 `addText` 做彩色文字
- [坐标与数学](position.md)——`Pos3D` / `Pos2D` / `BlockPosHelper`
- [实体](entities.md)——`EntityHelper`，配合追踪线和实体绑定
