---
icon: lucide/panel-top
---

# HUD 渲染

`Hud` 可以创建脚本屏幕，也可以注册 2D / 3D 绘制对象。常见用途：状态栏、坐标显示、实体标记、路径线。

## Draw2D 基础

```javascript
const d2d = Hud.createDraw2D()
d2d.addText("Hello JsMacros", 10, 10, 0xFFFFFF, true)
d2d.addRect(8, 8, 120, 24, 0x000000, 120)
d2d.register()
```

停止脚本时清理：

```javascript
context.unregisterOnStop(true, d2d)
```

或者：

```javascript
event.stopListener = JavaWrapper.methodToJava(() => d2d.unregister())
```

## 2D 元素

| 方法 | 作用 |
| --- | --- |
| `addText(text, x, y, color, shadow)` | 文字 |
| `addRect(x1, y1, x2, y2, color, alpha)` | 矩形 |
| `addLine(x1, y1, x2, y2, color)` | 线 |
| `addItem(x, y, id)` | 物品图标 |
| `addImage(...)` | 贴图 |
| `addDraw2D(draw, x, y, width, height)` | 嵌套 2D 绘制 |
| `addWorldPosWrapped(pos, base)` | 2.1.0：把 2D 元素包到世界坐标 |

!!! warning "zIndex 参数位置"
    很多 2D 方法有 `zIndex` 重载，例如 `addText(text, x, y, color, zIndex, shadow)`。别把 `zIndex` 和 `shadow` 写反。

## 更新文字

```javascript
const d2d = Hud.createDraw2D()
const line = d2d.addText("loading", 10, 10, 0xFFFFFF, true)
d2d.register()

JsMacros.on("Tick", JavaWrapper.methodToJava(() => {
    const p = Player.getPlayer()
    if (p) line.setText(`XYZ ${p.getX().toFixed(1)} ${p.getY().toFixed(1)} ${p.getZ().toFixed(1)}`)
}))
```

## Draw3D 基础

```javascript
const d3d = Hud.createDraw3D()
d3d.addBox(0, 64, 0, 1, 65, 1, 0x00FF00, 80, 0x00FF00, 30, true)
d3d.register()
```

常用方法：

| 方法 | 作用 |
| --- | --- |
| `addBox(x1, y1, z1, x2, y2, z2, color, fillColor, fill)` | 3D 方框 |
| `addLine(x1, y1, z1, x2, y2, z2, color)` | 3D 线 |
| `addTraceLine(x, y, z, color)` | 从玩家视角连到目标 |
| `addEntityTraceLine(entity, color)` | 连到实体 |
| `addPoint(x, y, z, radius, color)` | 小方块点 |
| `addDraw2D(x, y, z, ...)` | 世界里的 2D 面板 |
| `addEntityFollower(base, entity)` | 2.1.0：让元素跟随实体 |

## 注册与注销

```javascript
const draw = Hud.createDraw3D()
Hud.registerDraw3D(draw)
Hud.unregisterDraw3D(draw)
```

2D / 3D 对象也有自己的 `register()` / `unregister()`。新代码更推荐直接用对象方法。

## 纹理

```javascript
const image = Hud.createTexture("icons/example.png", "example_icon")
Chat.log(Hud.getRegisteredTextures().keySet())
```

创建后可以用 `addImage` 引用纹理 ID。

## 屏幕和鼠标信息

```javascript
Chat.log(Hud.getOpenScreenName())
Chat.log(`${Hud.getMouseX()}, ${Hud.getMouseY()}`)
Chat.log(`${Hud.getWindowWidth()}x${Hud.getWindowHeight()}`)
```

| 方法 | 作用 |
| --- | --- |
| `createScreen(title, dirtBG)` | 创建脚本屏幕 |
| `openScreen(screen)` | 打开屏幕，传 `null` 关闭 |
| `getOpenScreen()` | 当前屏幕 |
| `isContainer()` | 当前是否容器界面 |
| `getScaleFactor()` | GUI 缩放 |

