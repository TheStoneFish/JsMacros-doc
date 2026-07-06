---
icon: lucide/layout-dashboard
---

# 脚本屏幕

`Hud.createScreen(title, dirtBG)` 可以创建自定义 GUI。它适合做配置面板、快捷菜单和调试工具。

## 创建和打开

```javascript
const screen = Hud.createScreen("我的菜单", true)
Hud.openScreen(screen)
```

关闭：

```javascript
Hud.openScreen(null)
```

读取当前屏幕：

```javascript
const current = Hud.getOpenScreen()
Chat.log(Hud.getOpenScreenName())
```

## 和 HUD 的区别

| 能力 | 适合做 |
| --- | --- |
| `Draw2D` / `Draw3D` | 常驻显示、ESP、状态栏 |
| `ScriptScreen` | 可点击菜单、设置界面、输入框 |

## 回调写法

屏幕控件通常需要 Java 回调：

```javascript
const callback = JavaWrapper.methodToJava(() => {
    Chat.log("点击")
})
```

不同控件的 builder 和方法较多，建议打开类型提示查看 `ScriptScreen`、`ButtonWidgetHelper`、`TextFieldWidgetHelper`、`SliderWidgetHelper` 等类。

## 屏幕状态

```javascript
if (Hud.isContainer()) {
    Chat.log(Player.openInventory().getContainerTitle())
}
```

常用方法：

| 方法 | 作用 |
| --- | --- |
| `getOpenScreen()` | 当前屏幕对象 |
| `getOpenScreenName()` | 当前屏幕名 |
| `isContainer()` | 是否容器屏幕 |
| `getMouseX()` / `getMouseY()` | 鼠标坐标 |
| `getWindowWidth()` / `getWindowHeight()` | 窗口尺寸 |

!!! note "本页后续可扩展"
    脚本屏幕 API 类很多，本文先给入口和定位。具体控件页后续可以按按钮、文本框、滑条、列表分开补。

