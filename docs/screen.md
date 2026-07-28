---
icon: lucide/layout-dashboard
---

# 脚本屏幕（自定义 GUI）

`Hud.createScreen(title, dirtBG)` 可以创建自定义 GUI 屏幕（`ScriptScreen`，实现了 `IScreen` 接口）。它适合做配置面板、快捷菜单和调试工具：可以往上面放按钮、输入框、复选框、滑块、循环按钮，还能混入文字、矩形、物品图标等 2D 绘制元素。

## 和 HUD 的区别

| 能力 | 适合做 |
| --- | --- |
| `Draw2D` / `Draw3D` | 常驻显示、ESP、状态栏 |
| `ScriptScreen` | 可点击菜单、设置界面、输入框 |

`IScreen` 继承自 `IDraw2D<IScreen>`，也就是说 [HUD 渲染](hud.md) 里 `Draw2D` 的 `addText` / `addRect` / `addItem` / `addImage` 等方法在屏幕上同样可用，详见[与 Draw2D 元素混用](#draw2d)。

## 回调写法

屏幕控件的回调都要用 `JavaWrapper.methodToJava(...)` 包成 Java 可调用对象：

```javascript
const callback = JavaWrapper.methodToJava((btn, screen) => {
    Chat.log("点击")
})
```

控件回调的第一个参数是控件自身的 Helper（按钮回调收到 `ClickableWidgetHelper`、复选框回调收到 `CheckBoxWidgetHelper`……），第二个参数是屏幕 `IScreen`，方便直接在回调里改控件状态。

## 快速上手

一个带标题、输入框、复选框、按钮的设置界面。按钮点击会改变自己的文字，关闭（含 ESC）时自动保存：

```javascript
// 设置数据
const S = {
    name: "Steve",
    enabled: true,
    volume: 3
}

const screen = Hud.createScreen("示例设置", false) // false = 半透明黑背景，true = 泥土纹理背景

// 推荐把控件都加在 setOnInit 里：窗口缩放/reloadScreen 时会重新执行，控件不会丢
screen.setOnInit(JavaWrapper.methodToJava((scr) => {
    const cx = Math.floor(scr.getWidth() / 2)

    // 混用 Draw2D 元素：标题文字
    scr.addText("§l示例设置", cx - 30, 20, 0xFFFFFF, true)

    // 文本输入框：内容变化时回调
    const nameField = scr.addTextInput(cx - 100, 50, 200, 20, "输入名字…",
        JavaWrapper.methodToJava((text) => {
            S.name = text
        }))
    nameField.setText(S.name)

    // 复选框
    scr.addCheckbox(cx - 100, 80, 20, 20, "启用功能", S.enabled,
        JavaWrapper.methodToJava((cb) => {
            S.enabled = cb.isChecked()
        }))

    // 按钮：点击改变状态和自身文字
    scr.addButton(cx - 100, 110, 200, 20, `音量: ${S.volume}`,
        JavaWrapper.methodToJava((btn) => {
            S.volume = (S.volume % 5) + 1
            btn.setLabel(`音量: ${S.volume}`)
        }))

    // 关闭按钮
    scr.addButton(cx - 100, 140, 200, 20, "保存并关闭",
        JavaWrapper.methodToJava(() => {
            scr.close()
        }))
}))

// 关闭回调：无论点按钮关闭还是按 ESC 都会触发，适合在这里保存
screen.setOnClose(JavaWrapper.methodToJava(() => {
    FS.open("settings.json").write(JSON.stringify(S))
    Chat.log("设置已保存")
}))

Hud.openScreen(screen)
```

!!! tip "坐标系"
    屏幕使用 GUI 缩放后的坐标（受视频设置里的“GUI 缩放”影响），不是物理像素。用 `scr.getWidth()` / `scr.getHeight()` 拿当前屏幕宽高来做居中布局。

## 创建与打开（Hud 入口）

```javascript
const screen = Hud.createScreen("我的菜单", true)
Hud.openScreen(screen)
```

关闭（回到游戏）：

```javascript
Hud.openScreen(null)
```

读取当前屏幕：

```javascript
const current = Hud.getOpenScreen()
Chat.log(Hud.getOpenScreenName())
```

| 方法 | 返回 | 作用 |
| --- | --- | --- |
| `Hud.createScreen(title, dirtBG)` | `ScriptScreen` | 创建屏幕。`dirtBG` 为 `true` 用泥土纹理背景，`false` 用半透明背景 |
| `Hud.openScreen(s)` | `void` | 打开屏幕；传 `null` 关闭当前屏幕 |
| `Hud.getOpenScreen()` | `IScreen \| null` | 当前打开的屏幕对象（原版屏幕也会包成 `IScreen`） |
| `Hud.getOpenScreenName()` | `string \| null` | 当前屏幕名 |
| `Hud.isContainer()` | `boolean` | 当前屏幕是否是容器 |
| `Hud.getMouseX()` / `getMouseY()` | `number` | 鼠标坐标 |
| `Hud.getWindowWidth()` / `getWindowHeight()` | `number` | 当前窗口尺寸 |
| `Hud.getScaleFactor()` | `number` | 当前 GUI 缩放倍数 |
| `Hud.createTexture(width, height, name)` | `CustomImage` | 创建空白画布纹理，可用作屏幕背景、图片元素 |
| `Hud.createTexture(path, name)` | `CustomImage` | 从图片文件（绝对路径）创建纹理 |
| `Hud.getRegisteredTextures()` | `JavaMap` | 所有已注册的自定义纹理 |

### 屏幕状态

```javascript
if (Hud.isContainer()) {
    Chat.log(Player.openInventory().getContainerTitle())
}
```

## IScreen 完整参考

以下方法在 `Hud.createScreen` 返回的屏幕对象上都可用。所有 `add*` 方法会立即把控件加到屏幕上，并返回对应的控件 Helper 便于继续设置。

### 基本信息与控制

| 方法 | 返回 | 作用 |
| --- | --- | --- |
| `getScreenClassName()` | `string` | 屏幕类名 |
| `getTitleText()` | `TextHelper` | 屏幕标题 |
| `getWidth()` / `getHeight()` | `number` | 屏幕宽高（GUI 缩放坐标，继承自 `IDraw2D`） |
| `getButtonWidgets()` | `JavaList<ClickableWidgetHelper>` | 屏幕上所有按钮类控件（包括原版屏幕自带的） |
| `getTextFields()` | `JavaList<TextFieldWidgetHelper>` | 屏幕上所有文本输入框 |
| `close()` | `void` | 关闭屏幕（会触发 `setOnClose` 回调） |
| `reloadScreen()` | `IScreen` | 重新执行屏幕的 init（重建控件），见[常见坑](#常见坑) |
| `getOnClose()` | `MethodWrapper \| null` | 读取已设置的关闭回调 |

### addButton — 按钮

```
addButton(x, y, width, height, text, callback)
addButton(x, y, width, height, zIndex, text, callback)
```

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `x`, `y` | `int` | 位置 |
| `width`, `height` | `int` | 尺寸（原版按钮高度实际固定为 20） |
| `zIndex` | `int` | 层级（可选） |
| `text` | `string` | 按钮文字 |
| `callback` | `MethodWrapper` | 点击回调，参数 `(btn: ClickableWidgetHelper, screen: IScreen)` |

返回：`ClickableWidgetHelper`

```javascript
screen.addButton(10, 30, 100, 20, "点我", JavaWrapper.methodToJava((btn) => {
    btn.setLabel("点过了")
}))
```

### addTextInput — 文本输入框

```
addTextInput(x, y, width, height, message, onChange)
addTextInput(x, y, width, height, zIndex, message, onChange)
```

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `x`, `y`, `width`, `height` | `int` | 位置与尺寸 |
| `zIndex` | `int` | 层级（可选） |
| `message` | `string` | 占位提示文字 |
| `onChange` | `MethodWrapper` | 内容变化回调，参数 `(text: string, screen: IScreen)` |

返回：`TextFieldWidgetHelper`

```javascript
const input = screen.addTextInput(10, 60, 150, 20, "搜索…", JavaWrapper.methodToJava((text) => {
    Chat.log("当前输入: " + text)
}))
input.setMaxLength(64)
```

### addCheckbox — 复选框

```
addCheckbox(x, y, width, height, text, checked, callback)
addCheckbox(x, y, width, height, text, checked, showMessage, callback)
addCheckbox(x, y, width, height, zIndex, text, checked, callback)
addCheckbox(x, y, width, height, zIndex, text, checked, showMessage, callback)
```

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `x`, `y`, `width`, `height` | `int` | 位置与尺寸 |
| `zIndex` | `int` | 层级（可选） |
| `text` | `string` | 显示在复选框旁的文字 |
| `checked` | `boolean` | 初始是否勾选 |
| `showMessage` | `boolean` | 是否显示旁边的文字（可选，默认显示） |
| `callback` | `MethodWrapper` | 点击回调，参数 `(cb: CheckBoxWidgetHelper, screen: IScreen)` |

返回：`CheckBoxWidgetHelper`

### addSlider — 滑块

```
addSlider(x, y, width, height, text, value, callback)
addSlider(x, y, width, height, text, value, steps, callback)
addSlider(x, y, width, height, zIndex, text, value, callback)
addSlider(x, y, width, height, zIndex, text, value, steps, callback)
```

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `x`, `y`, `width`, `height` | `int` | 位置与尺寸 |
| `zIndex` | `int` | 层级（可选） |
| `text` | `string` | 显示在滑块里的文字 |
| `value` | `double` | 初始值 |
| `steps` | `int` | 档位数（可选，不填则为连续滑块） |
| `callback` | `MethodWrapper` | 值变化回调，参数 `(slider: SliderWidgetHelper, screen: IScreen)` |

返回：`SliderWidgetHelper`

```javascript
screen.addSlider(10, 90, 150, 20, "亮度", 0.5, 5, JavaWrapper.methodToJava((slider) => {
    slider.setLabel("亮度: " + slider.getValue())
}))
```

### addCyclingButton — 循环按钮

点一下切换到下一个值（类似原版“视野缩放：开/关”按钮）。

```
addCyclingButton(x, y, width, height, values, initial, callback)
addCyclingButton(x, y, width, height, zIndex, values, initial, callback)
addCyclingButton(x, y, width, height, zIndex, values, alternatives, initial, prefix, callback)
addCyclingButton(x, y, width, height, zIndex, values, alternatives, initial, prefix, alternateToggle, callback)
```

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `x`, `y`, `width`, `height` | `int` | 位置与尺寸 |
| `zIndex` | `int` | 层级（可选） |
| `values` | `string[]` | 循环的取值列表 |
| `alternatives` | `string[]` | 备选取值列表（可选，配合 `alternateToggle` 用） |
| `initial` | `string` | 初始值 |
| `prefix` | `string` | 显示前缀（如 `"模式"` 会显示成 `模式: 值`） |
| `alternateToggle` | `MethodWrapper` | 返回 `boolean`：`true` 时循环 `alternatives` 而不是 `values` |
| `callback` | `MethodWrapper` | 切换回调，参数 `(cyc: CyclingButtonWidgetHelper, screen: IScreen)` |

返回：`CyclingButtonWidgetHelper`

```javascript
screen.addCyclingButton(10, 120, 150, 20, ["低", "中", "高"], "中",
    JavaWrapper.methodToJava((cyc) => {
        Chat.log("当前档位: " + cyc.getStringValue())
    }))
```

### addLockButton — 锁形按钮

原版“锁定”图标按钮（世界设置里那种小锁），尺寸固定，不需要宽高参数。

```
addLockButton(x, y, callback)
addLockButton(x, y, zIndex, callback)
```

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `x`, `y` | `int` | 位置 |
| `zIndex` | `int` | 层级（可选） |
| `callback` | `MethodWrapper` | 点击回调，参数 `(lock: LockButtonWidgetHelper, screen: IScreen)` |

返回：`LockButtonWidgetHelper`

### 移除控件

| 方法 | 说明 |
| --- | --- |
| `removeElement(element)` | 移除任意元素（控件、文字、矩形都行，继承自 `IDraw2D`），推荐用这个 |
| `reAddElement(element)` | 把 `removeElement` 移除的元素加回来 |
| `removeButton(btn)` | 已弃用，用 `removeElement` |
| `removeTextInput(inp)` | 已弃用，用 `removeElement` |

## Builder 方式创建控件

除了 `add*` 方法，`IScreen` 还提供链式 Builder，参数多的时候更可读，还支持对齐布局。调用 `build()` 时创建控件并加到屏幕上。

| 入口方法 | 返回的 Builder | 用途 |
| --- | --- | --- |
| `buttonBuilder()` | `ButtonBuilder` | 按钮 |
| `texturedButtonBuilder()` | `TexturedButtonBuilder` | 贴图按钮 |
| `checkBoxBuilder()` / `checkBoxBuilder(checked)` | `CheckBoxBuilder` | 复选框 |
| `sliderBuilder()` | `SliderBuilder` | 滑块 |
| `textFieldBuilder()` | `TextFieldBuilder` | 文本输入框 |
| `cyclicButtonBuilder(valueToText)` | `CyclicButtonBuilder` | 循环按钮，`valueToText` 是把值转成 `TextHelper` 的回调 |
| `lockButtonBuilder()` / `lockButtonBuilder(locked)` | `LockButtonBuilder` | 锁形按钮 |

### 所有 Builder 通用的方法（AbstractWidgetBuilder）

| 方法 | 说明 |
| --- | --- |
| `x(x)` / `y(y)` / `pos(x, y)` | 位置；`getX()` / `getY()` 读取 |
| `width(w)` / `height(h)` / `size(w, h)` | 尺寸；`getWidth()` / `getHeight()` 读取（按钮类的高度固定 20，传了也会被忽略） |
| `zIndex(z)` / `getZIndex()` | 层级 |
| `message(text)` | 控件文字，接受 `string` 或 `TextHelper`；`getMessage()` 读取 |
| `active(bool)` / `isActive()` | 是否可交互（灰掉 = 不可点） |
| `visible(bool)` / `isVisible()` | 是否可见 |
| `alpha(a)` / `getAlpha()` | 不透明度（`double`） |
| `build()` | 创建控件、加到屏幕，返回对应 Helper |
| `alignHorizontally(...)` / `alignVertically(...)` / `align(...)` / `moveTo(x, y)` | 对齐布局（继承自 `Alignable`，见下方说明） |

### 各 Builder 特有的方法

| Builder | 方法 | 说明 |
| --- | --- | --- |
| `ButtonBuilder` | `action(cb)` / `getAction()` | 点击回调 `(btn: ButtonWidgetHelper, screen)` |
| `TexturedButtonBuilder` | `action(cb)` | 点击回调 |
|  | `enabledTexture(id)` / `disabledTexture(id)` | 正常/禁用状态贴图，接受资源 ID 字符串 |
|  | `enabledFocusedTexture(id)` / `disabledFocusedTexture(id)` | 悬停聚焦时的对应贴图 |
| `CheckBoxBuilder` | `checked(bool)` / `isChecked()` | 初始勾选状态 |
|  | `action(cb)` | 点击回调 `(cb: CheckBoxWidgetHelper, screen)` |
| `SliderBuilder` | `steps(n)` / `getSteps()` | 档位数，必须 ≥ 2 |
|  | `initially(value)` / `getValue()` | 初始档位，`0` 到 `steps - 1` 的整数 |
|  | `action(cb)` | 值变化回调 `(slider: SliderWidgetHelper, screen)` |
| `TextFieldBuilder` | `action(cb)` / `getAction()` | 内容变化回调 `(text: string, screen)` |
|  | `suggestion(text)` / `getSuggestion()` | 空白时的灰色提示文字 |
| `CyclicButtonBuilder` | `values(...vals)` | 取值列表（可变参数） |
|  | `values(defaults, alternatives)` | 同时给默认值和备选值（数组或 JavaList） |
|  | `alternatives(...vals)` | 备选取值列表 |
|  | `initially(value)` / `getInitialValue()` | 初始值 |
|  | `option(text)` / `getOption()` | 前缀文字（显示为 `前缀: 值`），接受 `string` 或 `TextHelper` |
|  | `omitTextOption(bool)` / `isOptionTextOmitted()` | 是否省略前缀 |
|  | `valueToText(cb)` / `getValueToText()` | 值转显示文字的回调，返回 `TextHelper` |
|  | `alternateToggle(cb)` / `getAlternateToggle()` | 返回 `true` 时循环备选值 |
|  | `action(cb)` / `getAction()` | 切换回调 `(cyc: CyclingButtonWidgetHelper, screen)` |
|  | `getDefaultValues()` / `getAlternateValues()` | 读取两组取值 |

### Builder 示例

```javascript
const screen = Hud.createScreen("Builder 示例", false)

screen.setOnInit(JavaWrapper.methodToJava((scr) => {
    const btn = scr.buttonBuilder()
        .pos(20, 40).width(120)
        .message("点我")
        .action(JavaWrapper.methodToJava(() => Chat.log("clicked")))
        .build()

    scr.textFieldBuilder()
        .pos(20, 70).size(120, 20)
        .suggestion("请输入…")
        .action(JavaWrapper.methodToJava((text) => Chat.log(text)))
        .build()

    scr.sliderBuilder()
        .pos(20, 100).size(120, 20)
        .steps(5).initially(2)
        .message("档位")
        .action(JavaWrapper.methodToJava((slider) => {
            slider.setLabel("档位: " + slider.getValue())
        }))
        .build()

    scr.cyclicButtonBuilder(JavaWrapper.methodToJava((v) =>
            Chat.createTextHelperFromString("模式: " + v)))
        .pos(20, 130).size(120, 20)
        .values("低", "中", "高")
        .initially("中")
        .action(JavaWrapper.methodToJava((cyc) => Chat.log("当前: " + cyc.getStringValue())))
        .build()

    // 对齐布局：把按钮水平居中
    btn.alignHorizontally("center")
}))

Hud.openScreen(screen)
```

## 控件 Helper 参考

`add*` 和 `build()` 返回的对象。所有控件共用 `ClickableWidgetHelper` 基类，各控件再补充自己的方法。大部分 setter 返回自身，可以链式调用。

### ClickableWidgetHelper（公共基类）

按钮 `addButton` 直接返回这个类型；其余控件 Helper 都继承它。

重要方法：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getX()` / `getY()` | `number` | 位置 |
| `setPos(x, y)` | 自身 | 移动位置 |
| `getWidth()` / `getHeight()` | `number` | 尺寸 |
| `setWidth(width)` | 自身 | 设置宽度 |
| `setLabel(text)` | 自身 | 改文字，接受 `string` 或 `TextHelper` |
| `getLabel()` | `TextHelper` | 当前文字 |
| `getActive()` / `setActive(bool)` | — | 是否可点（`false` 变灰） |
| `click()` / `click(await)` | 自身 | 用代码触发点击；`await` 为 `true` 时等点击处理完再返回 |
| `setTooltip(...tooltips)` | 自身 | 设置悬停提示（可传多行，接受字符串或 `TextHelper`） |
| `addTooltip(tooltip)` | 自身 | 追加一行提示 |
| `removeTooltip(index)` / `removeTooltip(textHelper)` | `boolean` | 删除提示 |
| `getTooltips()` | `JavaList<TextHelper>` | 提示副本 |

次要成员速查：

| 成员 | 说明 |
| --- | --- |
| `zIndex`（字段）/ `getZIndex()` | 层级 |
| `tooltips`（字段） | 底层提示列表 |
| `getScaledWidth()` / `getScaledHeight()` | 缩放后尺寸 |
| `getParentWidth()` / `getParentHeight()` | 父容器（屏幕）尺寸 |
| `getScaledLeft()` / `getScaledTop()` | 缩放后左/上边缘位置 |
| `moveTo(x, y)` | 移动（等价 `setPos`，来自 `Alignable`） |
| `ClickableWidgetHelper.clickedOn(screen)`（静态） | 内部辅助方法，一般用不到 |

!!! tip "对齐布局（Alignable）"
    控件 Helper 和 Builder 都实现了 `Alignable`，可以用 `alignHorizontally("center")`、`alignVertically("bottom", -10)`、`align("center", "center")` 相对屏幕对齐，或 `alignHorizontally(other, "LeftOnRight", 4)` 相对另一个控件对齐。格式为 `[left|center|right|x%]On[left|center|right|x%]`（垂直方向为 `top|center|bottom|y%`），大小写不敏感。

### ButtonWidgetHelper

普通按钮 / 贴图按钮（Builder 产物）。没有额外方法，全部功能见 `ClickableWidgetHelper`。

### TextFieldWidgetHelper（文本输入框）

重要方法：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getText()` | `string` | 当前内容 |
| `setText(text)` | 自身 | 设置内容 |
| `setText(text, await)` | 自身 | 设置内容，`await` 为 `true` 时等主线程处理完 |
| `setSuggestion(text)` | 自身 | 空白时的灰色提示文字 |
| `setMaxLength(length)` / `getMaxLength()` | — | 最大长度（默认 32，输入长内容前记得调大） |
| `setEditable(bool)` / `isEditable()` | — | 是否可编辑 |
| `setTextPredicate(cb)` | 自身 | 输入过滤器：回调收到字符串，返回 `true` 才允许 |
| `resetTextPredicate()` | 自身 | 清除过滤器 |

次要方法速查：

| 方法 | 说明 |
| --- | --- |
| `setEditableColor(color)` / `setUneditableColor(color)` | 可编辑/不可编辑状态的文字颜色 |
| `getSelectedText()` | 当前选中的文字 |
| `setSelection(start, end)` | 设置选区 |
| `setCursorPosition(pos)` / `setCursorPosition(pos, shift)` | 移动光标（`shift` 为 `true` 时扩展选区） |
| `setCursorToStart()` / `setCursorToStart(shift)` | 光标移到开头 |
| `setCursorToEnd()` / `setCursorToEnd(shift)` | 光标移到末尾 |

### CheckBoxWidgetHelper（复选框）

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isChecked()` | `boolean` | 是否勾选 |
| `setChecked(bool)` | 自身 | 设置勾选状态 |
| `toggle()` | 自身 | 反转勾选状态 |

### SliderWidgetHelper（滑块）

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getValue()` | `number` | 当前值 |
| `setValue(value)` | 自身 | 设置值（`double`） |
| `getSteps()` / `setSteps(steps)` | — | 档位数 |

### CyclingButtonWidgetHelper（循环按钮）

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getValue()` | 值 | 当前值 |
| `getStringValue()` | `string` | 当前值的字符串形式 |
| `setValue(val)` | `boolean` | 设置值，值变了返回 `true` |
| `forward()` / `backward()` | 自身 | 切到下一个 / 上一个值 |
| `cycle(amount)` | 自身 | 前进 `amount` 个值（负数后退） |

### LockButtonWidgetHelper（锁形按钮）

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `isLocked()` | `boolean` | 是否锁定 |
| `setLocked(bool)` | 自身 | 设置锁定状态 |

## 鼠标/键盘事件回调

这些方法设置屏幕级的输入回调，返回 `IScreen` 自身可链式调用：

| 方法 | 回调参数 | 触发时机 |
| --- | --- | --- |
| `setOnMouseDown(cb)` | `(pos: Pos2D, button: int)` | 鼠标按下。`button`：0 左键 / 1 右键 / 2 中键 |
| `setOnMouseDrag(cb)` | `(vec: Vec2D, button: int)` | 按住拖动，`Vec2D` 是拖动向量（起点到终点） |
| `setOnMouseUp(cb)` | `(pos: Pos2D, button: int)` | 鼠标松开 |
| `setOnScroll(cb)` | `(pos: Pos2D, delta: Pos2D)` | 滚轮滚动。2.1.0 里第二个参数是 `Pos2D`（`x` 横向、`y` 纵向滚动量）；旧版本是 `double` |
| `setOnKeyPressed(cb)` | `(key: int, mods: int)` | 按键按下，`key` 是 GLFW 键码，`mods` 是修饰键位掩码 |
| `setOnCharTyped(cb)` | `(char, mods: int)` | 输入字符（区别于键码，适合处理文字输入） |
| `setOnInit(cb)` | `(screen: IScreen)` | 屏幕初始化（打开、窗口缩放、`reloadScreen` 都会触发） |
| `setOnFailInit(cb)` | `(error: string)` | `setOnInit` 回调抛异常时触发 |
| `setOnClose(cb)` | `(screen: IScreen)` | 屏幕关闭（含 ESC） |

```javascript
const screen = Hud.createScreen("画板", false)

screen.setOnMouseDrag(JavaWrapper.methodToJava((vec, btn) => {
    // 沿拖动轨迹画一条线
    screen.addLine(vec.x1, vec.y1, vec.x2, vec.y2, 0xFFFF0000)
}))

screen.setOnKeyPressed(JavaWrapper.methodToJava((key, mods) => {
    Chat.log(`键码 ${key}, 修饰键 ${mods}`)
}))

Hud.openScreen(screen)
```

## ScriptScreen 特有成员

`Hud.createScreen` 实际返回的 `ScriptScreen` 在 `IScreen` 之外还有：

| 成员 | 类型 | 说明 |
| --- | --- | --- |
| `drawTitle` | `boolean` 字段 | 是否绘制屏幕标题 |
| `shouldCloseOnEsc` | `boolean` 字段 | 是否允许 ESC 关闭 |
| `shouldPause` | `boolean` 字段 | 打开时是否暂停游戏（单人） |
| `setParent(parent)` | 方法 | 设置父屏幕：本屏幕关闭后自动回到父屏幕（接受 `IScreen` 或原版 `Screen`） |
| `setOnRender(cb)` | 方法 | 在主线程的渲染函数里加自定义逻辑。回调参数 `(pos: Pos3D, ctx)`，`pos` 的三个分量是 `mouseX`、`mouseY`、`tickDelta`，`ctx` 是原版 `GuiGraphics` |

!!! warning "shouldCloseOnEsc = false 有风险"
    d.ts 原文警告：如果设成 `false` 又没提供别的关闭途径（比如关闭按钮里调用 `screen.close()`），玩家会被卡在屏幕里出不来。

!!! warning "setOnRender 运行在主线程"
    每帧都会调用。回调里绝对不要 `Time.sleep`、`Client.waitTick` 或做重活，否则整个游戏卡死。只做轻量绘制/读数。

## 与 Draw2D 元素混用

`IScreen extends IDraw2D<IScreen>`，所以 [HUD 渲染](hud.md) 里的 2D 元素方法全部可以直接在屏幕上用：

| 方法 | 作用 |
| --- | --- |
| `addText(text, x, y, color, shadow)` | 文字（另有带 `zIndex` / `scale` / `rotation` 的重载） |
| `addRect(x1, y1, x2, y2, color, alpha)` | 矩形 |
| `addLine(x1, y1, x2, y2, color)` | 线 |
| `addItem(x, y, id)` | 物品图标 |
| `addImage(...)` | 贴图（可配合 `Hud.createTexture`） |
| `addDraw2D(draw, x, y, width, height)` | 嵌套一个 Draw2D |
| `getElements()` / `removeElement(e)` / `reAddElement(e)` | 管理所有元素（控件也算元素） |

给控件区加个半透明底板：

```javascript
screen.setOnInit(JavaWrapper.methodToJava((scr) => {
    scr.addRect(15, 30, 185, 160, 0x000000, 120)  // 颜色 0x000000，不透明度 120
    // 带层级的重载是 addRect(x1, y1, x2, y2, color, alpha, rotation, zIndex)，注意 zIndex 在 rotation 之后
    scr.addText("标题", 20, 35, 0xFFFF55, true)
    scr.addItem(20, 50, "minecraft:diamond")
    scr.addButton(20, 130, 160, 20, "确定", JavaWrapper.methodToJava(() => scr.close()))
}))
```

各元素的详细参数、`Text` / `Rect` / `Image` 对象的方法见 [HUD 渲染](hud.md)。

## 常见坑 {#常见坑}

!!! warning "窗口缩放会清掉直接添加的控件"
    Minecraft 在窗口大小或 GUI 缩放变化时会重新执行屏幕的 init。**不在 `setOnInit` 回调里添加的控件/元素这时会全部消失。**所以正式脚本一律把控件添加逻辑写进 `setOnInit`，让它随时可重建。

!!! note "reloadScreen 的用途"
    `reloadScreen()` 手动触发一次 init 重建。典型用法：配置数据变了、想按新状态重新生成整个界面时，改完数据调一次 `reloadScreen()`，比逐个更新控件省事。

!!! warning "回调运行在哪个线程"
    控件回调由游戏主线程触发，经 `MethodWrapper` 进入你的脚本上下文执行。同一个 JS 上下文同时只能跑一段代码：脚本主体还没跑完时回调会排队；回调里调用 `Client.waitTick()`、`Time.sleep()` 这类等待方法可能阻塞主线程甚至死锁。回调里要做耗时操作（写文件、网络请求）时，用 `JavaWrapper.methodToJavaAsync(...)` 包装，让它异步执行。

!!! warning "ESC 关闭与清理"
    玩家按 ESC 也会触发 `setOnClose` 回调，所以“保存设置”这类清理逻辑写在 `setOnClose` 里最保险，不要只写在“保存”按钮里。屏幕被 `Hud.openScreen(null)` 或另一个屏幕顶掉时同理。

!!! note "按钮点了没反应？"
    控件回调依赖脚本上下文存活。如果你在 JsMacros 界面手动停止了脚本（或服务被关掉），屏幕还开着但所有回调都已失效。重新运行脚本再打开屏幕即可。
