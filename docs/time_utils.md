---
icon: lucide/clock
---

# 时间与工具

`Time`、`Utils` 和 `JavaUtils` 是杂项工具库。它们不直接操作游戏世界，但写脚本经常会用到。

## Time

```javascript
const start = Time.time()
Time.sleep(1000)
Chat.log(`耗时 ${Time.time() - start}ms`)
```

| 方法 | 作用 |
| --- | --- |
| `time()` | 当前时间戳，毫秒 |
| `sleep(millis)` | 阻塞当前脚本线程 |

!!! warning "sleep 不是 waitTick"
    `Time.sleep()` 按真实时间睡眠；`Client.waitTick()` 按游戏 tick 等待。游戏内循环更推荐 `Client.waitTick()`。

## Utils

```javascript
Chat.log(Utils.hashString("hello"))
Chat.log(Utils.encode("hello"))
Chat.log(Utils.decode(Utils.encode("hello")))
```

| 方法 | 作用 |
| --- | --- |
| `hashString(message)` | 默认哈希 |
| `hashString(message, algorithm)` | 指定算法 |
| `hashString(message, algorithm, base64)` | 指定输出 |
| `encode(message)` / `decode(message)` | Base64 编解码 |
| `requireNonNull(obj)` | 空值断言 |
| `guessName(text)` | 猜测名称 |
| `guessNameAndRoles(text)` | 猜测名称和角色 |

## JavaUtils

创建 Java 集合：

```javascript
const list = JavaUtils.createArrayList()
list.add("a")
list.add("b")

const map = JavaUtils.createHashMap()
map.put("key", "value")
```

随机数：

```javascript
const random = JavaUtils.getRandom()
Chat.log(random.nextInt(100))
```

其他工具：

| 方法 | 作用 |
| --- | --- |
| `createHashSet()` | Java HashSet |
| `getRandom(seed)` | 带种子的随机数 |
| `getHelperFromRaw(raw)` | 原始 Minecraft 对象转 helper，可能返回 `null` |
| `arrayToString(array)` | 数组转字符串 |
| `arrayDeepToString(array)` | 深层数组转字符串 |

## JavaWrapper

事件、HUD 回调、遍历函数经常要把 JS 函数包装成 Java 可调用对象。

```javascript
JsMacros.on("Tick", JavaWrapper.methodToJava((e, context) => {
    // ...
}))
```

| 方法 | 作用 |
| --- | --- |
| `methodToJava(fn)` | 同步包装 |
| `methodToJavaAsync(fn)` | 异步包装 |
| `methodToJavaAsync(priority, fn)` | 指定优先级 |
| `deferCurrentTask()` | 延后当前任务 |
| `getCurrentPriority()` | 当前优先级 |
| `stop()` | 停止当前脚本 |

