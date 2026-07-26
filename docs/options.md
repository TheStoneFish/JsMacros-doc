---
icon: lucide/settings-2
---

# 游戏设置

`OptionsHelper` 让脚本读写 Minecraft 的游戏设置——视频、声音、按键、聊天、皮肤、辅助功能，基本对应 ESC → "选项" 里能点到的东西。

通过 [客户端](client.md) 库获取：

```javascript
const opts = Client.getGameOptions()
```

## 结构总览

`OptionsHelper` 本体管一些通用设置（语言、难度、视野、资源包等），具体分类拆在 6 个子 helper 里。子 helper 既是只读字段也有对应 getter，两种写法等价：

```javascript
const opts = Client.getGameOptions()

opts.video            // 或 opts.getVideoOptions()
opts.music            // 或 opts.getMusicOptions()
opts.control          // 或 opts.getControlOptions()
opts.chat             // 或 opts.getChatOptions()
opts.skin             // 或 opts.getSkinOptions()
opts.accessibility    // 或 opts.getAccessibilityOptions()
```

| 字段 / getter | 类型 | 管什么 |
| --- | --- | --- |
| `video` / `getVideoOptions()` | `VideoOptionsHelper` | 帧率、视距、画质、亮度、全屏 |
| `music` / `getMusicOptions()` | `MusicOptionsHelper` | 各类音量、音频设备、字幕 |
| `control` / `getControlOptions()` | `ControlOptionsHelper` | 鼠标、自动跳跃、按键绑定（只读） |
| `chat` / `getChatOptions()` | `ChatOptionsHelper` | 聊天显示、宽高、命令建议 |
| `skin` / `getSkinOptions()` | `SkinOptionsHelper` | 皮肤图层、主手 |
| `accessibility` / `getAccessibilityOptions()` | `AccessibilityOptionsHelper` | 旁白、字幕、切换潜行/疾跑等 |

每个子 helper 都有 `parent` 字段和 `getParent()` 方法指回主 `OptionsHelper`，所以链式调用可以跨分类：

```javascript
Client.getGameOptions().video.setMaxFps(120).getParent().saveOptions()
```

!!! warning "改完记得 saveOptions"
    所有 setter 只改内存中的值。**`saveOptions()` 会把设置写入 `options.txt`**，不调用的话有些改动重启就丢，部分选项甚至不立刻生效。养成习惯：一串 set 之后补一个 `saveOptions()`。

!!! tip "sendSyncedOptions：让服务器知道你改了设置"
    皮肤图层、（服务端可见的）语言这类设置需要同步给服务器。正常游戏里这发生在关闭设置界面时，但脚本不开界面，所以要手动调 `opts.sendSyncedOptions()`（2.0.0+）。

## OptionsHelper 主类

### 保存与同步

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `saveOptions()` | 自身 | 保存设置到磁盘 |
| `sendSyncedOptions()` | `void` | 把需要同步的设置发给服务器 |

### 资源包

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getResourcePacks()` | `JavaList<string>` | 所有资源包名 |
| `getEnabledResourcePacks()` | `JavaList<string>` | 已启用的资源包名 |
| `setEnabledResourcePacks(enabled: string[])` | 自身 | 设置启用的资源包列表 |
| `removeServerResourcePack(state: boolean)` | `OptionsHelper` | `true` 移除服务器资源包，`false` 放回去 |

```javascript
const opts = Client.getGameOptions()
Chat.log(`可用资源包: ${opts.getResourcePacks()}`)
Chat.log(`已启用: ${opts.getEnabledResourcePacks()}`)
```

### 语言与难度

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getLanguage()` | `Locale` | 当前语言代码（如 `zh_cn`） |
| `setLanguage(languageCode)` | 自身 | 切换语言 |
| `getDifficulty()` | `Difficulty` | 当前难度 |
| `setDifficulty(name)` | 自身 | 设置难度，取值见下方枚举表 |
| `isDifficultyLocked()` | `boolean` | 难度是否已锁定 |
| `lockDifficulty()` | 自身 | 锁定难度 |
| `unlockDifficulty()` | 自身 | 解锁难度（原版客户端做不到，慎用） |

### 视野与镜头

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getFov()` / `setFov(fov: int)` | `number` / 自身 | 视野角度 |
| `getCameraMode()` / `setCameraMode(mode)` | `Trit` / `OptionsHelper` | 视角：`0` 第一人称，`1` 第三人称背后，`2` 第三人称正面 |
| `getSmoothCamera()` / `setSmoothCamera(val: boolean)` | `boolean` / `OptionsHelper` | 电影视角（平滑镜头） |

### 窗口尺寸

| 方法 | 说明 |
| --- | --- |
| `getWidth()` / `getHeight()` | 当前窗口宽 / 高（像素） |
| `setWidth(w)` / `setHeight(h)` / `setSize(w, h)` | 调整窗口尺寸 |

### 已废弃的旧方法

这些方法还能用，但都有更清晰的替代品，新脚本别再写：

| 旧方法 | 改用 |
| --- | --- |
| `getCloudMode()` / `setCloudMode(mode)` | `video.getCloudsMode()` / `setCloudsMode(mode)` |
| `getGraphicsMode()` / `setGraphicsMode(mode)` | `video.getGraphicsMode()` / `setGraphicsMode(mode)` |
| `getRenderDistance()` / `setRenderDistance(d)` | `video.getRenderDistance()` / `setRenderDistance(radius)` |
| `getGamma()` / `setGamma(gamma)` | `video.getGamma()` / `setGamma(gamma)` |
| `getGuiScale()` / `setGuiScale(scale)` | `video.getGuiScale()` / `setGuiScale(scale)` |
| `isRightHanded()` / `setRightHanded(val)` | `skin.isRightHanded()` / `toggleMainHand(hand)` |
| `setVolume(vol)` | `music.setMasterVolume(volume)` |
| `setVolume(category, volume)` | `music.setVolume(category, volume)` |
| `getVolume(category)` / `getVolumes()` | `music.getVolume(category)` / `getVolumes()` |

## 视频设置 VideoOptionsHelper

读写成对列出（`get`/`is` 读，`set`/`enable` 写，写方法均返回自身可链式）：

| 读取 | 设置 | 说明 |
| --- | --- | --- |
| `getMaxFps()` | `setMaxFps(maxFps: int)` | 最大帧率 |
| `getRenderDistance()` | `setRenderDistance(radius: int)` | 视距（区块） |
| `getSimulationDistance()` | `setSimulationDistance(radius: int)` | 模拟距离（区块） |
| `getGraphicsMode()` | `setGraphicsMode(mode)` | 图形品质，`GraphicsMode` |
| `getCloudsMode()` | `setCloudsMode(mode)` | 云渲染，`CloudsMode` |
| `getParticleMode()` | `setParticleMode(mode)` | 粒子效果，`ParticleMode` |
| `getChunkBuilderMode()` | `setChunkBuilderMode(mode)` | 区块构建器，`ChunkBuilderMode` |
| `getSmoothLightningMode()` | `setSmoothLightningMode(mode: boolean)` | 平滑光照 |
| `isVsyncEnabled()` | `enableVsync(val: boolean)` | 垂直同步 |
| `isFullscreen()` | `setFullScreen(fullscreen: boolean)` | 全屏 |
| `getFullscreenResolution()` | —（只读） | 全屏分辨率字符串 |
| `getGuiScale()` | `setGuiScale(scale: int)` | GUI 缩放，1–4 |
| `getGamma()` | `setGamma(gamma: double)` | 亮度 gamma，正常范围 0–1 |
| `getBrightness()` | `setBrightness(gamma: double)` | 亮度 |
| `getBiomeBlendRadius()` | `setBiomeBlendRadius(radius: int)` | 生物群系混合半径 |
| `getMipMapLevels()` | `setMipMapLevels(val: int)` | Mipmap 级别 |
| `isViewBobbingEnabled()` | `enableViewBobbing(val: boolean)` | 视角摇晃 |
| `areEntityShadowsEnabled()` | `enableEntityShadows(val: boolean)` | 实体阴影 |
| `getEntityDistance()` | `setEntityDistance(val: double)` | 实体渲染距离 |
| `getAttackIndicatorType()` | `setAttackIndicatorType(type)` | 攻击指示器，`AttackIndicatorType` |
| `getDistortionEffect()` | `setDistortionEffects(val: double)` | 屏幕扭曲效果强度 |
| `getFovEffects()` | `setFovEffects(val: double)` | FOV 效果强度 |
| `isAutosaveIndicatorEnabled()` | `enableAutosaveIndicator(val: boolean)` | 自动保存指示器 |

```javascript
// 常用示例：录像前压低帧率上限、拉远视距
const video = Client.getGameOptions().video
video.setMaxFps(60)
     .setRenderDistance(16)
     .setGraphicsMode("fancy")
     .getParent().saveOptions()
Chat.log(`FPS 上限 ${video.getMaxFps()}, 视距 ${video.getRenderDistance()}`)
```

## 声音设置 MusicOptionsHelper

| 读取 | 设置 | 说明 |
| --- | --- | --- |
| `getMasterVolume()` | `setMasterVolume(volume: double)` | 主音量 |
| `getMusicVolume()` | `setMusicVolume(volume: double)` | 音乐 |
| `getRecordsVolume()` | `setRecordsVolume(volume: double)` | 唱片机/音符盒 |
| `getWeatherVolume()` | `setWeatherVolume(volume: double)` | 天气 |
| `getBlocksVolume()` | `setBlocksVolume(volume: double)` | 方块 |
| `getHostileVolume()` | `setHostileVolume(volume: double)` | 敌对生物 |
| `getNeutralVolume()` | `setNeutralVolume(volume: double)` | 友好生物 |
| `getPlayerVolume()` | `setPlayerVolume(volume: double)` | 玩家 |
| `getAmbientVolume()` | `setAmbientVolume(volume: double)` | 环境 |
| `getVoiceVolume()` | `setVoiceVolume(volume: double)` | 声音/语音 |
| `getVolume(category)` | `setVolume(category, volume)` | 按分类读写，`SoundCategory` |
| `getVolumes()` | — | 所有分类音量的 `JavaMap<string, number>` |
| `getSoundDevice()` | `setSoundDevice(audioDevice: string)` | 当前音频输出设备 |
| `getAudioDevices()` | — | 可用音频设备列表 |
| `areSubtitlesShown()` | `showSubtitles(val: boolean)` | 字幕开关 |

音量都是 `0.0`（静音）到 `1.0`（最大）的小数。

```javascript
// 挂机时把音乐关掉、主音量降到 30%
Client.getGameOptions().music
    .setMusicVolume(0)
    .setMasterVolume(0.3)
    .getParent().saveOptions()
```

## 控制设置 ControlOptionsHelper

### 鼠标与移动

| 读取 | 设置 | 说明 |
| --- | --- | --- |
| `getMouseSensitivity()` | `setMouseSensitivity(val: double)` | 鼠标灵敏度 |
| `isMouseInverted()` | `invertMouse(val: boolean)` | 鼠标反转 |
| `getMouseWheelSensitivity()` | `setMouseWheelSensitivity(val: double)` | 滚轮灵敏度 |
| `isDiscreteScrollingEnabled()` | `enableDiscreteScrolling(val: boolean)` | 离散滚动（修复某些系统滚太快） |
| `isRawMouseInputEnabled()` | `enableRawMouseInput(val: boolean)` | 原始鼠标输入 |
| `isTouchscreenEnabled()` | `enableTouchscreen(val: boolean)` | 触屏模式 |
| `isAutoJumpEnabled()` | `enableAutoJump(val: boolean)` | 自动跳跃 |
| `isSneakTogglingEnabled()` | `toggleSneak(val: boolean)` | 潜行改为切换式 |
| `isSprintTogglingEnabled()` | `toggleSprint(val: boolean)` | 疾跑改为切换式 |

```javascript
// 常用示例：关掉自动跳跃
Client.getGameOptions().control.enableAutoJump(false).getParent().saveOptions()
```

### 按键绑定（只读）

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getRawKeys()` | `JavaArray<any>` | 原始 `KeyMapping` 对象数组 |
| `getCategories()` | `JavaList<KeyCategory>` | 所有按键分类 |
| `getKeys()` | `JavaList<Key>` | 所有按键名 |
| `getKeyBinds()` | `JavaMap<Bind, Key>` | 每个动作 → 绑定的按键 |
| `getKeyBindsByCategory(category)` | `JavaMap<string, string>` | 指定分类下的绑定 |
| `getKeyBindsByCategory()` | `JavaMap<string, JavaMap<string, string>>` | 按分类分组的全部绑定 |

动作名形如 `key.attack`、`key.jump`，按键名形如 `key.keyboard.f`、`key.mouse.left`。

```javascript
// 查一下攻击键现在绑在哪
const binds = Client.getGameOptions().control.getKeyBinds()
Chat.log(`攻击键: ${binds.get("key.attack")}`)
```

!!! note "OptionsHelper 不能改按键绑定"
    `ControlOptionsHelper` 只提供按键绑定的**读取**，没有对应 setter。想在脚本里模拟按键、监听按键，用 [按键](keybind.md) 库（`KeyBind`）。

## 聊天设置 ChatOptionsHelper

| 读取 | 设置 | 说明 |
| --- | --- | --- |
| `getChatVisibility()` | `setChatVisibility(mode)` | 聊天可见性，`ChatVisibility` |
| `areColorsShown()` | `setShowColors(val: boolean)` | 允许消息里的颜色代码 |
| `areWebLinksEnabled()` | `enableWebLinks(val: boolean)` | 允许打开聊天里的网页链接 |
| `isWebLinkPromptEnabled()` | `enableWebLinkPrompt(val: boolean)` | 打开链接前弹确认框 |
| `getChatOpacity()` | `setChatOpacity(val: double)` | 聊天不透明度 |
| `getTextBackgroundOpacity()` | `setTextBackgroundOpacity(val: double)` | 文字背景不透明度 |
| `getTextSize()` | `setTextSize(val: double)` | 文字大小 |
| `getChatLineSpacing()` | `setChatLineSpacing(val: double)` | 行距 |
| `getChatDelay()` | `setChatDelay(val: double)` | 聊天延迟（秒） |
| `getChatWidth()` | `setChatWidth(val: double)` | 聊天框宽度 |
| `getChatFocusedHeight()` | `setChatFocusedHeight(val: double)` | 聚焦时高度 |
| `getChatUnfocusedHeight()` | `setChatUnfocusedHeight(val: double)` | 未聚焦时高度 |
| `getNarratorMode()` | `setNarratorMode(mode)` | 旁白模式，`NarratorMode` |
| `areCommandSuggestionsEnabled()` | `enableCommandSuggestions(val: boolean)` | 命令建议 |
| `areMatchedNamesHidden()` | `enableHideMatchedNames(val: boolean)` | 隐藏屏蔽用户的消息 |
| `isDebugInfoReduced()` | `reduceDebugInfo(val: boolean)` | 简化调试信息（F3） |

```javascript
// 直播场景：只显示系统消息，防止刷屏
Client.getGameOptions().chat.setChatVisibility("SYSTEM").getParent().saveOptions()
```

## 皮肤设置 SkinOptionsHelper

| 读取 | 设置 | 说明 |
| --- | --- | --- |
| `isCapeActivated()` | `toggleCape(val: boolean)` | 披风 |
| `isJacketActivated()` | `toggleJacket(val: boolean)` | 外套 |
| `isLeftSleeveActivated()` | `toggleLeftSleeve(val: boolean)` | 左袖 |
| `isRightSleeveActivated()` | `toggleRightSleeve(val: boolean)` | 右袖 |
| `isLeftPantsActivated()` | `toggleLeftPants(val: boolean)` | 左裤腿 |
| `isRightPantsActivated()` | `toggleRightPants(val: boolean)` | 右裤腿 |
| `isHatActivated()` | `toggleHat(val: boolean)` | 帽子层 |
| `isRightHanded()` / `isLeftHanded()` | `toggleMainHand(hand: string)` | 主手，`hand` 只能是 `"left"` 或 `"right"` |

```javascript
// 全开皮肤图层并同步给服务器，让别人也看得到
const opts = Client.getGameOptions()
opts.skin.toggleCape(true).toggleJacket(true).toggleHat(true)
opts.sendSyncedOptions()
opts.saveOptions()
```

## 辅助功能 AccessibilityOptionsHelper

| 读取 | 设置 | 说明 |
| --- | --- | --- |
| `getNarratorMode()` | `setNarratorMode(mode: string)` | 旁白模式，取值同 `NarratorMode` |
| `areSubtitlesShown()` | `showSubtitles(val: boolean)` | 字幕 |
| `getTextBackgroundOpacity()` | `setTextBackgroundOpacity(val: double)` | 文字背景不透明度 |
| `isBackgroundForChatOnly()` | `enableBackgroundForChatOnly(val: boolean)` | 背景只用于聊天 |
| `getChatOpacity()` | `setChatOpacity(val: double)` | 聊天不透明度 |
| `getChatLineSpacing()` | `setChatLineSpacing(val: double)` | 聊天行距 |
| `getChatDelay()` | `setChatDelay(val: double)` | 聊天延迟（秒） |
| `isAutoJumpEnabled()` | `enableAutoJump(val: boolean)` | 自动跳跃 |
| `isSneakTogglingEnabled()` | `toggleSneak(val: boolean)` | 切换式潜行 |
| `isSprintTogglingEnabled()` | `toggleSprint(val: boolean)` | 切换式疾跑 |
| `getDistortionEffect()` | `setDistortionEffect(val: double)` | 扭曲效果强度 |
| `getFovEffect()` | `setFovEffect(val: double)` | FOV 效果强度 |
| `isMonochromeLogoEnabled()` | `enableMonochromeLogo(val: boolean)` | 单色 Logo |
| `areLightningFlashesHidden()` | — | 是否隐藏闪电闪光（只读） |

一部分选项（自动跳跃、切换潜行/疾跑、聊天外观）和控制、聊天分类是同一份设置的两个入口，改哪边效果一样。

!!! note "d.ts 里的一个小坑"
    `setFovEffect` 在 d.ts 中有一个多出来的 `boolean` 重载，紧跟在 `areLightningFlashesHidden()` 之后，疑似是"隐藏闪电闪光"的 setter 被错误命名。传 `double` 的版本才是正经的 FOV 效果设置。

## 枚举字符串速查

这些参数类型本质都是固定字符串（或小整数），传错会不生效或报错：

| 类型 | 可取值 | 用在 |
| --- | --- | --- |
| `Difficulty` | `"peaceful"` `"easy"` `"normal"` `"hard"` | `setDifficulty` |
| `GraphicsMode` | `"fast"` `"fancy"` `"fabulous"` | `video.setGraphicsMode` |
| `CloudsMode` | `"off"` `"fast"` `"fancy"` | `video.setCloudsMode` |
| `ParticleMode` | `"minimal"` `"decreased"` `"all"` | `video.setParticleMode` |
| `ChunkBuilderMode` | `"none"` `"nearby"` `"player_affected"` | `video.setChunkBuilderMode` |
| `AttackIndicatorType` | `"off"` `"crosshair"` `"hotbar"` | `video.setAttackIndicatorType` |
| `ChatVisibility` | `"FULL"` `"SYSTEM"` `"HIDDEN"` | `chat.setChatVisibility` |
| `NarratorMode` | `"OFF"` `"ALL"` `"CHAT"` `"SYSTEM"` | `chat.setNarratorMode`、`accessibility.setNarratorMode` |
| `Trit` | `0` `1` `2` | `setCameraMode`（0 第一人称，2 正面） |
| `SoundCategory` | 原版分类：`"master"` `"music"` `"record"` `"weather"` `"block"` `"hostile"` `"neutral"` `"player"` `"ambient"` `"voice"` | `music.getVolume` / `setVolume` |

!!! tip "SoundCategory 等类型来自生成文件"
    `SoundCategory`、`Key`、`Bind`、`KeyCategory`、`Locale` 这些类型定义在自动生成的 `McIdsAndEnums.d.ts` 里（用 `Client.generateTypescriptIdsAndEnums()` 生成，见 [客户端](client.md)），上表列的是原版取值。

## 完整示例：一键"性能模式"

```javascript
// 把画质相关设置压到最低，再存盘；适合卡顿时救急
const opts = Client.getGameOptions()

opts.video
    .setGraphicsMode("fast")
    .setCloudsMode("off")
    .setParticleMode("minimal")
    .setRenderDistance(8)
    .setSimulationDistance(8)
    .setMaxFps(60)
    .enableVsync(false)
    .enableEntityShadows(false)

opts.music.setMusicVolume(0)
opts.saveOptions()

Chat.log("§a性能模式已开启")
```
