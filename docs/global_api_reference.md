---
icon: lucide/list
---

# 全局库参考

这一页集中列出 `JsMacros-2.1.0.d.ts` 里直接注入脚本的全局库方法。想学习怎么用，先看各专题页；想确认方法名，来这里查。

## Chat

| 方法 | 作用 |
| --- | --- |
| `log(message)` / `log(message, await)` | 本地输出 |
| `logf(message, ...args)` | 格式化本地输出 |
| `logColor(message)` | 颜色文本输出 |
| `say(message)` | 发送聊天或命令 |
| `sayf(message, ...args)` | 格式化发送 |
| `open(message)` | 打开聊天框并预填 |
| `title(title, subtitle, fadeIn, remain, fadeOut)` | 显示标题 |
| `actionbar(text)` / `actionbar(text, tinted)` | 显示 ActionBar |
| `toast(title, desc)` / `toast(title, desc, displayTimeMs)` | 显示 Toast |
| `createTextHelperFromString(content)` | 创建文本 helper |
| `createTextHelperFromTranslationKey(key, ...content)` | 翻译键文本 |
| `createTextHelperFromJSON(json)` | JSON 文本，失败返回 `null` |
| `createTextBuilder()` | 文本构造器 |
| `createCommandBuilder(name)` | 命令构造器 |
| `unregisterCommand(name)` / `reRegisterCommand(node)` | 命令注册管理 |
| `getCommandManager()` | 命令管理器 |
| `getHistory()` | 聊天历史 |
| `getTextWidth(text)` | 文本宽度 |
| `sectionSymbolToAmpersand(string)` | `§` 转 `&` |
| `ampersandToSectionSymbol(string)` | `&` 转 `§` |
| `stripFormatting(string)` | 去格式 |
| `getLogger()` / `getLogger(name)` | logger |

## Client

| 方法 | 作用 |
| --- | --- |
| `getMinecraft()` | 原始 Minecraft 客户端 |
| `getRegistryManager()` | 注册表 helper |
| `createPacketByteBuffer()` | 创建 packet buffer |
| `runOnMainThread(runnable, ...)` | 主线程执行 |
| `getGameOptions()` | 游戏设置 |
| `mcVersion()` | Minecraft 版本 |
| `getFPS()` | FPS |
| `loadWorld(folderName)` | 加载单人世界 |
| `connect(ip)` / `connect(ip, port)` | 连接服务器 |
| `disconnect()` / `disconnect(callback)` | 断开连接 |
| `shutdown()` | 关闭客户端 |
| `waitTick()` / `waitTick(i)` | 等待 tick |
| `ping(ip)` / `pingAsync(ip, callback)` | 服务器 ping |
| `cancelAllPings()` | 取消 ping |
| `getLoadedMods()` / `isModLoaded(modId)` / `getMod(modId)` | Mod 信息 |
| `grabMouse()` | 捕获鼠标 |
| `isDevEnv()` | 是否开发环境 |
| `getModLoader()` | Mod 加载器 |
| `getRegisteredBlocks()` / `getRegisteredItems()` | 注册表内容 |
| `exitGamePeacefully()` / `exitGameForcefully()` | 退出游戏 |
| `sendPacket(packet)` / `receivePacket(packet)` | 包操作 |
| `getClipboard()` / `setClipboard(text)` | 剪贴板 |
| `generateTypescriptIdsAndEnums(...)` | 生成类型 ID 和枚举 |

## FS

| 方法 | 作用 |
| --- | --- |
| `list(path)` | 列目录 |
| `exists(path)` | 是否存在 |
| `isDir(path)` / `isFile(path)` | 类型判断 |
| `getName(path)` / `getDir(path)` | 文件名/目录 |
| `toRelativePath(absolutePath)` | 转相对路径 |
| `createFile(path, name, createDirs?)` | 创建文件 |
| `makeDir(path)` | 创建目录 |
| `move(from, to)` / `copy(from, to)` | 移动/复制 |
| `unlink(path)` | 删除 |
| `combine(patha, pathb)` | 拼路径 |
| `open(path)` / `open(path, charset)` | 打开文件 |
| `walkFiles(path, maxDepth, followLinks, visitor)` | 遍历文件 |
| `toRawFile(path)` / `toRawPath(path)` | Java 原始对象 |
| `getRawAttributes(path)` | 文件属性 |

## GlobalVars

| 方法 | 作用 |
| --- | --- |
| `putInt` / `putString` / `putDouble` / `putBoolean` / `putObject` | 写值 |
| `getInt` / `getString` / `getDouble` / `getBoolean` / `getObject` | 读值 |
| `getType(name)` | 类型 |
| `getAndIncrementInt(name)` / `incrementAndGetInt(name)` | 自增 |
| `getAndDecrementInt(name)` / `decrementAndGetInt(name)` | 自减 |
| `toggleBoolean(name)` | 布尔取反 |
| `remove(key)` | 删除 |
| `getRaw()` | 原始 Map |

## Hud

| 方法 | 作用 |
| --- | --- |
| `createScreen(title, dirtBG)` | 创建屏幕 |
| `openScreen(screen)` | 打开/关闭屏幕 |
| `getOpenScreen()` / `getOpenScreenName()` | 当前屏幕 |
| `createTexture(width, height, name)` / `createTexture(path, name)` | 创建纹理 |
| `getRegisteredTextures()` | 纹理表 |
| `getScaleFactor()` | GUI 缩放 |
| `isContainer()` | 当前是否容器 |
| `createDraw2D()` / `createDraw3D()` | 创建绘制对象 |
| `registerDraw2D` / `unregisterDraw2D` / `listDraw2Ds` / `clearDraw2Ds` | 2D 管理 |
| `registerDraw3D` / `unregisterDraw3D` / `listDraw3Ds` / `clearDraw3Ds` | 3D 管理 |
| `getMouseX()` / `getMouseY()` | 鼠标坐标 |
| `getWindowWidth()` / `getWindowHeight()` | 窗口尺寸 |

## JavaUtils / JavaWrapper

| 方法 | 作用 |
| --- | --- |
| `JavaUtils.createArrayList(array?)` | Java ArrayList |
| `JavaUtils.createHashMap()` | Java HashMap |
| `JavaUtils.createHashSet()` | Java HashSet |
| `JavaUtils.getRandom(seed?)` | 随机数 |
| `JavaUtils.getHelperFromRaw(raw)` | raw 转 helper |
| `JavaUtils.arrayToString(array)` / `arrayDeepToString(array)` | 数组字符串化 |
| `JavaWrapper.methodToJava(fn)` | JS 函数转 Java 回调 |
| `JavaWrapper.methodToJavaAsync(fn)` | 异步回调 |
| `JavaWrapper.deferCurrentTask(priorityAdjust?)` | 延后当前任务 |
| `JavaWrapper.getCurrentPriority()` | 当前优先级 |
| `JavaWrapper.stop()` | 停止脚本 |

## JsMacros

| 方法 | 作用 |
| --- | --- |
| `getProfile()` / `getConfig()` | 配置 |
| `getServiceManager()` | 服务管理器 |
| `getOpenContexts()` | 运行中的上下文 |
| `runScript(...)` | 运行脚本文件或字符串 |
| `wrapScriptRun(...)` / `wrapScriptRunAsync(...)` | 包装脚本运行 |
| `open(path)` / `openUrl(url)` | 打开文件/URL，部分入口已弃用 |
| `on(event, ...)` / `once(event, ...)` | 注册事件 |
| `off(listener)` / `off(event, listener)` | 取消监听 |
| `disableAllListeners(event?)` | 关闭监听 |
| `disableScriptListeners(event?)` | 关闭当前脚本监听 |
| `waitForEvent(event, ...)` | 等待事件 |
| `listeners(event)` | 监听列表 |
| `eventFilters()` | 事件过滤器 |
| `createCustomEvent(eventName)` | 自定义事件 |
| `assertEvent(event, type)` | 类型断言 |

## KeyBind

| 方法 | 作用 |
| --- | --- |
| `getKeyCode(keyName)` | 取按键码 |
| `getKeyBindings()` | 原版按键绑定表 |
| `setKeyBind(bind, key)` | 修改绑定 |
| `key(keyName, keyState)` | 直接按键 |
| `pressKey(keyName)` / `releaseKey(keyName)` | 按下/释放按键 |
| `keyBind(keyBind, keyState)` | 操作绑定 |
| `pressKeyBind(keyBind)` / `releaseKeyBind(keyBind)` | 按下/释放绑定 |
| `getPressedKeys()` | 当前按下的键 |

## Player

| 方法 | 作用 |
| --- | --- |
| `openInventory()` | 背包/容器 |
| `getPlayer()` | 本地玩家，可能 `null` |
| `getInteractionManager()` / `interactions()` | 交互管理器 |
| `getGameMode()` / `setGameMode(gameMode)` | 游戏模式 |
| `rayTraceBlock(distance, fluid)` | 方块射线 |
| `detailedRayTraceBlock(distance, fluid)` | 详细方块射线 |
| `rayTraceEntity(distance?)` | 实体射线 |
| `writeSign(...)` | 写告示牌 |
| `takeScreenshot(...)` / `takePanorama(...)` | 截图 |
| `getStatistics()` | 统计 |
| `getReach()` | 触及距离 |
| `createPlayerInput(...)` / `createPlayerInputsFromCsv/Json(...)` | 移动输入 |
| `getCurrentPlayerInput()` / `addInput` / `addInputs` / `clearInputs` | 输入队列 |
| `setDrawPredictions(val)` | 绘制预测 |
| `predictInput` / `predictInputs` | 移动预测 |
| `isBreakingBlock()` | 是否挖方块，已建议用交互管理器 |
| `moveForward` / `moveBackward` / `moveStrafeLeft` / `moveStrafeRight` | 移动辅助 |
| `createVec` / `createLookingVector` / `createPos` / `createBlockPos` | 坐标向量 |

## PositionCommon

| 方法 | 作用 |
| --- | --- |
| `createPos(x, y, z)` / `createPos(x, y)` | 创建 `Pos3D` / `Pos2D` |
| `createVec(x1, y1, z1, x2, y2, z2)` / `createVec(x1, y1, x2, y2)` | 创建 `Vec3D` / `Vec2D` |
| `createLookingVector(entity)` / `createLookingVector(yaw, pitch)` | 视线方向向量 |
| `createBlockPos(x, y, z)` | 创建 `BlockPosHelper` |

详见 [位置与向量](position.md)。

## Reflection

| 方法 | 作用 |
| --- | --- |
| `getClass(name)` | Java 类 |
| `getDeclaredMethod` / `getMethod` | 方法 |
| `getDeclaredField` / `getField` | 字段 |
| `invokeMethod(m, obj, ...objects)` | 调用 |
| `newInstance(c, ...objects)` | 创建实例 |
| `createClassProxyBuilder` / `createClassBuilder` | 动态类 |
| `getClassFromClassBuilderResult(cName)` | 取构建结果 |
| `createLibraryBuilder` / `createLibrary` | 创建库 |
| `compileJavaClass` / `getCompiledJavaClass` / `getAllCompiledJavaClassVersions` | 编译类 |
| `getReflect(obj)` | jOOR Reflect |
| `loadJarFile(file)` | 加载 jar |
| `getClassName(o)` | 类名 |

## Request

| 方法 | 作用 |
| --- | --- |
| `create(url)` | 创建 HTTPRequest |
| `get(url, headers?)` | GET |
| `post(url, data, headers?)` | POST |
| `createWS(url)` / `createWS2(url)` | WebSocket |

## Time / Utils

| 方法 | 作用 |
| --- | --- |
| `Time.time()` | 毫秒时间 |
| `Time.sleep(millis)` | 睡眠 |
| `Utils.guessName(text)` / `guessNameAndRoles(text)` | 名称猜测 |
| `Utils.hashString(message, algorithm?, base64?)` | 哈希 |
| `Utils.encode(message)` / `decode(message)` | Base64 |
| `Utils.requireNonNull(obj, message?)` | 非空断言 |

## World

| 方法 | 作用 |
| --- | --- |
| `isWorldLoaded()` | 世界是否加载 |
| `getLoadedPlayers()` / `getPlayers()` / `getPlayerEntry(name)` | 玩家 |
| `getBlock(...)` / `getChunk(x, z)` | 方块/区块 |
| `getWorldScanner(...)` | 世界扫描器 |
| `findBlocksMatching(...)` | 搜索方块 |
| `iterateSphere(...)` / `iterateBox(...)` | 遍历区域 |
| `getScoreboards()` | 计分板 |
| `getEntities(...)` | 实体 |
| `rayTraceBlock(...)` / `rayTraceEntity(...)` | 世界射线 |
| `getDimension()` / `getBiome()` / `getBiomeAt(...)` | 维度/群系 |
| `getTime()` / `getTimeOfDay()` / `isDay()` / `isNight()` | 时间 |
| `isRaining()` / `isThundering()` | 天气 |
| `getWorldIdentifier()` | 世界标识 |
| `getRespawnPos()` | 重生点 |
| `getDifficulty()` / `getMoonPhase()` | 难度/月相 |
| `getSkyLight(x, y, z)` / `getBlockLight(x, y, z)` | 光照 |
| `playSoundFile(file, volume)` / `playSound(...)` | 声音 |
| `getBossBars()` | BossBar |
| `isChunkLoaded(chunkX, chunkZ)` | 区块是否加载 |
| `getCurrentServerAddress()` | 当前服务器地址 |
| `getServerTPS()` | 服务器 TPS 估算 |

