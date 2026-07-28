---
icon: lucide/image
---

# 自定义图像（CustomImage）

`CustomImage` 是一张放在内存里的位图画布（内部是 `java.awt.image.BufferedImage`），创建时会自动注册为 Minecraft 纹理。你可以在上面逐像素画点、画线、画几何图形、贴图、写字，然后通过 `getIdentifier()` 把它当成普通纹理用在 HUD、脚本屏幕的 `Image` 元素里——相当于给脚本一块可编程的"画板"。

典型用途：动态小地图、自绘图表/进度环、给加载的图片打水印、运行时生成的图标或背景。

!!! note "本页与《HUD 渲染》的分工"
    `Image` 元素本身（`addImage` 的 14 个参数、`imageBuilder()` 的全部链式方法）在 [HUD 渲染](hud.md#image) 已完整讲过。本页专讲 `CustomImage` 对象：怎么创建、怎么在上面画、怎么让修改生效。

## 工作流程：四步

```
创建纹理 → 在画布上绘制 → update() 提交 → 用 getIdentifier() 显示
```

1. **创建**：`Hud.createTexture(...)` 新建空白画布或从图片文件加载。
2. **绘制**：调用本页的各种 `draw*` / `fill*` / `setPixel` 方法。所有绘图方法都返回自身，可以链式调用。
3. **提交**：调用 `update()`，把内存里的修改上传为游戏纹理。**不调用 `update()`，屏幕上永远是旧画面。**
4. **显示**：把 `getIdentifier()` 交给 `Draw2D` / 屏幕的 `Image` 元素，或直接用 `imageBuilder().fromCustomImage(img)`。

## 创建：Hud.createTexture

`Hud` 命名空间提供两个重载（均自 1.8.4 起）：

| 重载 | 返回 | 说明 |
| --- | --- | --- |
| `Hud.createTexture(width, height, name)` | `CustomImage` | 新建一张 `width x height` 的空白画布 |
| `Hud.createTexture(path, name)` | `CustomImage` | 从图片文件加载，`path` 是**绝对路径** |

- `name` 可以传 `null`；建议给一个唯一名称，方便之后在 `Hud.getRegisteredTextures()`（返回 `JavaMap<string, CustomImage>`，所有已注册纹理的只读 Map）里找回它。
- 返回的对象即已注册为游戏纹理，无需额外注册步骤。

```javascript
// 新建 128x64 空白画布
const canvas = Hud.createTexture(128, 64, "my_canvas")

// 从磁盘加载图片（注意：这里要绝对路径）
const logo = Hud.createTexture("E:/图片/logo.png", "my_logo")
```

!!! warning "三个方法、三种路径规则"
    | 方法 | 路径基准 |
    | --- | --- |
    | `Hud.createTexture(path, name)` | **绝对路径** |
    | `img.loadImage(path, ...)` | 相对 **jsMacros 配置文件夹** |
    | `img.saveImage(path, fileName)` | 相对 **jsMacros 配置文件夹** |

    混用是最常见的"文件找不到"原因。

??? info "进阶：静态方法与构造器"
    `CustomImage` 类上还有等价的静态入口，一般脚本用 `Hud.createTexture` 即可：

    | 成员 | 说明 |
    | --- | --- |
    | `CustomImage.createWidget(width, height, name)` | 同新建空白画布 |
    | `CustomImage.createWidget(path, name)` | 从文件加载，失败时返回 `null` |
    | `CustomImage.IMAGES` | 静态只读 `JavaMap<string, CustomImage>`，等价 `Hud.getRegisteredTextures()` |
    | `new CustomImage(bufferedImage)` / `new CustomImage(bufferedImage, name)` | 直接用 Java 的 `BufferedImage` 构造 |

## 颜色格式：ARGB、RGB 与 ABGR

CustomImage 里出现三种颜色格式，别混淆：

| 场景 | 格式 | 例子 |
| --- | --- | --- |
| `getPixel` / `setPixel` | **ARGB**（含透明度） | 不透明红 `0xFFFF0000` |
| `setGraphicsColor`（画笔颜色，管所有 `draw*`/`fill*`） | **RGB** | 红 `0xFF0000` |
| `nativeARGBFlip(argb)`（静态工具） | ARGB → **ABGR** | 见下 |

`CustomImage.nativeARGBFlip(argb)` 是静态颜色工具：Minecraft 原生纹理内部使用 ABGR 格式，此方法把 ARGB 颜色翻转成 ABGR。只有直接操作原生纹理数据时才用得到，日常绘图不需要。

!!! tip "在 JavaScript 里拼 ARGB"
    用位运算拼出来的结果是带符号 32 位整数，可直接传给 Java 的 `int` 参数：
    ```javascript
    const argb = (0xFF << 24) | (r << 16) | (g << 8) | b
    ```

## update()：让修改生效

> 将纹理更新为当前图像内容。对图像做的任何修改，**只有调用本方法后才会显示出来**。不必每改一笔就调用一次，而应该在整轮修改完成后调用。（译自 JSDoc）

```javascript
img.fillRect(0, 0, 32, 32)
   .drawLine(0, 0, 63, 63)
   .setPixel(10, 10, 0xFFFFFFFF)
   .update()   // 一轮改完，提交一次
```

`update()` 返回自身，自 1.8.4 起。这是本类**唯一**的刷新入口，忘了调它是"画了却看不见"的头号原因。

## 方法参考

以下按功能分组，签名与 `JsMacros-2.1.0.d.ts` 逐一对应。除特别注明外，方法均自 1.8.4 起、返回自身（可链式）。

### 基本信息

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getName()` | `string` | 图像名称 |
| `getIdentifier()` | `string` | 纹理标识符，交给 Draw2D 等一切需要 identifier 的地方 |
| `getWidth()` | `number` | 宽度。**创建后是常量，不会变** |
| `getHeight()` | `number` | 高度。**创建后是常量，不会变** |
| `getImage()` | `java.awt.image.BufferedImage` | 内部 BufferedImage（所有修改的落点），可交给其他 Java API |

尺寸不可变意味着：想"扩容"只能新建一张更大的 CustomImage，再用 `drawImage` 把旧图画上去。

### 像素读写

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getPixel(x, y)` | `number` | 读取该点颜色（**ARGB**） |
| `setPixel(x, y, argb)` | 自身 | 写入该点颜色（**ARGB**） |

```javascript
const img = Hud.createTexture(16, 16, "pixel_demo")
// 画一条对角线上的白点
for (let i = 0; i < 16; i++) {
    img.setPixel(i, i, (0xFF << 24) | 0xFFFFFF)
}
img.update()
Chat.log("(0,0) 的颜色: " + (img.getPixel(0, 0) >>> 0).toString(16))
```

### 画笔状态与裁剪

所有 `draw*` / `fill*` 几何与文字方法都使用同一支"画笔"，先设置颜色再画：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `setGraphicsColor(color)` | 自身 | 设置画笔颜色（**RGB**），影响之后所有 draw/fill 操作 |
| `getGraphicsColor()` | `number` | 当前画笔颜色（RGB） |
| `translate(x, y)` | 自身 | 平移坐标原点，之后的绘制坐标都相对新原点 |
| `clipRect(x, y, width, height)` | 自身 | 用给定矩形与当前裁剪区**求交**，绘制不会超出裁剪区 |
| `setClip(x, y, width, height)` | 自身 | 直接**替换**裁剪区为给定矩形 |
| `getClipBounds()` | `java.awt.Rectangle` | 当前裁剪区边界（有 `x`/`y`/`width`/`height` 字段） |
| `setPaintMode()` | 自身 | 恢复普通覆盖绘制模式 |
| `setXorMode(color)` | 自身 | 切换为 XOR 模式：绘制时与给定颜色做异或。同一图形画两次会还原，可做简易反色/闪烁效果；用完记得 `setPaintMode()` 恢复 |

```javascript
img.setGraphicsColor(0x00FF00)     // 之后画的都是绿色
   .setClip(0, 0, 32, 32)          // 只允许画在左上角 32x32 区域
   .fillRect(0, 0, 64, 64)         // 实际只有 32x32 被填充
   .update()
```

### 几何绘制

规律：`draw*` 画轮廓线，`fill*` 画实心，颜色都来自 `setGraphicsColor`。

| 方法 | 说明 |
| --- | --- |
| `drawLine(x1, y1, x2, y2)` | 两点之间画直线 |
| `drawRect(x, y, width, height)` / `fillRect(...)` | 矩形轮廓 / 实心矩形 |
| `clearRect(x, y, width, height)` | 清空矩形区域 |
| `clearRect(x, y, width, height, color)` | 用给定 RGB 颜色清空（填充）矩形区域 |
| `drawRoundRect(x, y, width, height, arcWidth, arcHeight)` / `fillRoundRect(...)` | 圆角矩形，`arcWidth`/`arcHeight` 是四角圆弧的水平/垂直直径 |
| `draw3DRect(x, y, width, height, raised)` / `fill3DRect(...)` | 带高光阴影的"3D"矩形，`raised` 为 `true` 凸起、`false` 凹陷 |
| `drawOval(x, y, width, height)` / `fillOval(...)` | 椭圆（内切于给定矩形）。**画圆就是 width == height 的椭圆**，没有单独的 drawCircle |
| `drawArc(x, y, width, height, startAngle, arcAngle)` / `fillArc(...)` | 圆弧/扇形：`startAngle` 起始角，`arcAngle` 相对起始角扫过的角度 |
| `drawPolygonLine(pointsX, pointsY)` | 折线（依次连接各点，**不闭合**） |
| `drawPolygon(pointsX, pointsY)` | 多边形轮廓（自动闭合） |
| `fillPolygon(pointsX, pointsY)` | 实心多边形 |
| `copyArea(x, y, width, height, dx, dy)` | 把区域 `(x, y, width, height)` 复制到偏移 `(dx, dy)` 处（偏移量而非目标坐标） |

多边形的 `pointsX` / `pointsY` 是两个 int 数组，长度必须相同、顺序一一对应。

!!! tip "角度约定（AWT）"
    圆弧角度单位是度：0 度指向三点钟方向，正角度为逆时针。`drawArc(0, 0, 64, 64, 0, 90)` 画的是右上角四分之一圆弧。

```javascript
const shapes = Hud.createTexture(96, 96, "shapes_demo")
shapes.setGraphicsColor(0xFF5555)
      .fillOval(4, 4, 40, 40)                      // 实心圆
      .setGraphicsColor(0x55FF55)
      .drawRoundRect(50, 4, 40, 40, 10, 10)        // 圆角矩形轮廓
      .setGraphicsColor(0x5555FF)
      .fillArc(4, 50, 40, 40, 90, 270)             // 四分之三扇形
      .setGraphicsColor(0xFFFF55)
      .fillPolygon([70, 90, 50], [50, 90, 90])     // 三角形
      .update()
```

### 图像操作：加载、绘制、复制、保存

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `loadImage(path)` | `BufferedImage` 或 `null` | 从磁盘读一张图（路径相对 jsMacros 配置文件夹），**不会画上去**，供 `drawImage` 使用 |
| `loadImage(path, x, y, width, height)` | `BufferedImage` | 同上，但只取图中 `(x, y, width, height)` 的子区域 |
| `drawImage(img, x, y, width, height)` | 自身 | 把 `img` 画到本画布 `(x, y)` 处，并**缩放**到 `width x height` |
| `drawImage(img, x, y, width, height, sourceX, sourceY, sourceWidth, sourceHeight)` | 自身 | 只画 `img` 的子区域 `(sourceX, sourceY, sourceWidth, sourceHeight)`，缩放到目标 `width x height` |
| `saveImage(path, fileName)` | 自身 | 保存为 **png**。`path` 相对 jsMacros 配置文件夹，`fileName` 不带扩展名 |

`drawImage` 的第一个参数是 `java.awt.Image`，三个常见来源：

- `img.loadImage(...)` 读磁盘文件；
- 另一张 CustomImage 的 `getImage()`——实现画布之间互相拼贴；
- 本画布自己的 `getImage()`——配合不同目标尺寸实现自缩放。

```javascript
// 拼贴 + 缩放 + 存盘
const board = Hud.createTexture(128, 128, "collage")
const icon = board.loadImage("icons/star.png")        // <配置文件夹>/icons/star.png
if (icon) {
    board.drawImage(icon, 0, 0, 128, 128)             // 铺满整张画布（缩放）
         .drawImage(icon, 96, 96, 32, 32)             // 右下角再贴一个小的
}
board.copyArea(0, 0, 64, 64, 64, 0)                   // 左上 64x64 复制到右上
     .update()
     .saveImage("out", "collage_result")              // <配置文件夹>/out/collage_result.png
```

### 文字绘制

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `drawString(x, y, text)` | 自身 | 在 `(x, y)` 处用当前画笔颜色画文字 |
| `getStringWidth(toAnalyze)` | `number` | 该字符串按当前字体的像素宽度，用于对齐/居中计算 |

!!! warning "关于字体"
    d.ts 中没有暴露任何设置字体的方法，`drawString` 使用 Java AWT 的默认字体。另外按 AWT 约定，`y` 是文字**基线**位置而不是文字顶部——想让文字贴着某个顶边，`y` 要往下加一个字高。需要花哨字体时，建议在图片编辑器里做好再 `loadImage` / `drawImage` 贴进来。

```javascript
const label = Hud.createTexture(100, 20, "label_demo")
const text = "Hello JsMacros"
const w = label.getStringWidth(text)
label.setGraphicsColor(0xFFFFFF)
     .drawString(Math.floor((100 - w) / 2), 14, text)   // 水平居中
     .update()
```

## 在 Draw2D / 屏幕里显示

`getIdentifier()` 返回的字符串可以用在一切需要纹理 identifier 的地方。两种写法：

```javascript
const tex = Hud.createTexture(64, 64, "demo")
// ...绘制并 update() ...

const d2d = Hud.createDraw2D()
d2d.setOnInit(JavaWrapper.methodToJava(() => {
    // 写法 1（推荐）：fromCustomImage 自动填好 identifier、区域、纹理尺寸
    d2d.imageBuilder().fromCustomImage(tex).pos(10, 10).size(64, 64).buildAndAdd()

    // 写法 2：手动传 identifier（后 6 个参数：纹理内采样区 x, y, w, h + 纹理总宽高）
    d2d.addImage(90, 10, 64, 64, tex.getIdentifier(), 0, 0, 64, 64, 64, 64)
}))
d2d.register()
```

`addImage` 全部重载、`Image$Builder` 全部方法、以及 Draw2D 的生命周期规则见 [HUD 渲染](hud.md#image)。在 `Hud.createScreen()` 创建的屏幕上用法完全相同（`IScreen` 同样有 `addImage` / `imageBuilder`），见[脚本屏幕](screen.md)。

## 完整示例：渐变色块 + 文字显示在 HUD

64x64 渐变 + 白色边框 + 居中文字，注册为纹理显示在屏幕左上角。可直接整段粘贴运行：

```javascript
// 1. 创建 64x64 空白画布
const img = Hud.createTexture(64, 64, "gradient_demo")

// 2. 逐像素画红蓝渐变（ARGB：A=255，R 随 x 增大，B 随 y 增大）
for (let x = 0; x < 64; x++) {
    for (let y = 0; y < 64; y++) {
        const r = Math.floor(x / 63 * 255)
        const b = Math.floor(y / 63 * 255)
        img.setPixel(x, y, (0xFF << 24) | (r << 16) | (0x40 << 8) | b)
    }
}

// 3. 白色边框 + 居中文字（画笔颜色是 RGB）
const text = "JsM"
const textW = img.getStringWidth(text)
img.setGraphicsColor(0xFFFFFF)
   .drawRect(0, 0, 63, 63)
   .drawString(Math.floor((64 - textW) / 2), 38, text)

// 4. 提交修改——没有这步屏幕上是全透明的空纹理
img.update()

// 5. 注册 Draw2D 显示（元素创建放 onInit，窗口缩放后依然存在）
const d2d = Hud.createDraw2D()
d2d.setOnInit(JavaWrapper.methodToJava(() => {
    d2d.imageBuilder()
        .fromCustomImage(img)
        .pos(10, 10)
        .size(64, 64)
        .buildAndAdd()
}))
d2d.register()

// 提示：普通脚本结束后 d2d 会一直留在屏幕上。
// 想常驻请做成服务并 event.unregisterOnStop(true, d2d)，
// 临时测试可以稍后手动执行 d2d.unregister()。
```

## 性能与注意事项

- **攒一批再 `update()`**。JSDoc 明确说明：不必每改一笔就调用，整轮修改结束后调用一次即可。`update()` 会把整张图重新上传为纹理，放在每 tick 高频循环里逐像素改一次就 update 一次是典型的性能陷阱。
- **逐像素操作本身不便宜**。大面积填充优先用 `fillRect` / `clearRect(x, y, w, h, color)`，而不是双重循环 `setPixel`。
- **尺寸是常量**。`getWidth()` / `getHeight()` 创建后不会变；需要更大画布只能新建并用 `drawImage` 迁移内容。
- **纹理注册是全局的**。CustomImage 一经创建就注册进 `Hud.getRegisteredTextures()`，脚本结束也不会消失；给 `name` 起唯一名字，重复运行脚本时优先复用已注册的对象而不是无限新建。
- **画完没反应先查两件事**：忘了 `update()`；或者画笔/像素颜色的透明度为 0（ARGB 高 8 位是 0 时完全透明）。
- **Draw2D 窗口缩放会清空元素**，但 CustomImage 纹理本身不受影响——把 `imageBuilder`/`addImage` 放进 `setOnInit` 就够了，不需要重新绘制画布。

## 相关页面

- [HUD 渲染](hud.md)——`Image` 元素、`addImage` 全部重载、`Image$Builder` 速查、Draw2D 生命周期
- [脚本屏幕](screen.md)——把 CustomImage 用作可交互屏幕上的图片/背景
- [文件系统](fs.md)——配置文件夹内文件的读写与路径管理
- [Java 互操作](java_api.md)——`BufferedImage` 等 AWT 类型的进一步操作
