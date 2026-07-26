---
icon: lucide/code-2
---

# Java 与反射

JsMacros 脚本运行在 JVM 里：JS 版脚本由 **GraalJS** 引擎执行，脚本里拿到的很多对象本来就是 Java 对象，可以直接调用它们的 Java 方法。本页讲的是最底层的三件事：**怎么拿 Java 类**、**怎么把 JS 函数交给 Java 调用**（`JavaWrapper` / `MethodWrapper`）、**怎么用反射**（`Reflection`）。

普通脚本优先用 Helper（见[常用类型](types.md)）；只有 Helper 不够用时再进入这一层。如果目标是调用其他 Mod 暴露的 Java API，比如 Baritone，先看[外部 API 总览](external_api.md)，再看 [Baritone API](baritone_api.md)。

## GraalJS 与 Java 互操作基础

先建立三个直觉：

1. **Java 对象直接用。** `Player.getPlayer()` 返回的 Helper、`Client.getMinecraft()` 返回的原始客户端对象，都是真正的 Java 对象，直接 `obj.method()` 调用即可。GraalJS 会自动做基本类型转换（JS number ↔ Java int/double、JS string ↔ Java String 等，细节见[常用类型](types.md)）。
2. **JS 函数不能直接当 Java 回调。** 凡是参数类型标注为 `MethodWrapper` 的 API（`JsMacros.on`、`Client.runOnMainThread`、Java 集合的 `forEach` 等），都必须先用 `JavaWrapper` 包装，原因见下一节。
3. **`new` 一个 Java 类就像 new 一个 JS 类。** `new Packages.java.util.ArrayList()` 直接可用。

### 三种取 Java 类的方式

| 方式 | 来源 | 编辑器补全 | 说明 |
| --- | --- | --- | --- |
| `Packages.java.util.ArrayList` | JsMacros 全局 `Packages` 包树 | 有（d.ts 已声明的类） | 首选。按包路径逐级访问 |
| `Reflection.getClass("java.util.ArrayList")` | JsMacros `Reflection` 库 | 有 | 支持"双名"回退，适合 Minecraft 混淆类（见下文 Reflection 一节） |
| `Java.type("java.util.ArrayList")` | GraalJS 引擎内置 | 无 | GraalJS 运行时可用，但 d.ts 没有声明它，编辑器会报错；且其他脚本语言（Python/Lua 等）没有这个函数。文档统一用前两种 |

```javascript
// 三种写法在 JS 运行时是等价的
const ArrayList1 = Packages.java.util.ArrayList
const ArrayList2 = Reflection.getClass("java.util.ArrayList")

const list = new ArrayList1()
list.add("hello")
Chat.log(list.size()) // 1
```

拿到类之后，`.class` 静态属性返回对应的 `java.lang.Class` 对象（反射时经常要用它当参数类型）：

```javascript
const stringClass = Packages.java.lang.String.class
Chat.log(stringClass.getName()) // java.lang.String
```

### Packages 树里有什么

d.ts 中 `declare namespace Packages`（3424 行起）声明了这些顶层包：

| 包 | 内容 |
| --- | --- |
| `Packages.java.*` | JDK 标准库：`java.lang`、`java.util`、`java.io`、`java.time`、`java.text`、`java.nio` 等 |
| `Packages.xyz.wagyourtail.*` | JsMacros 自己的类：`MethodWrapper`、各种 Helper、`ProxyBuilder` 等 |
| `Packages.com.google.gson.*` | Gson（`JsonObject`、`JsonArray` 等 JSON 处理） |
| `Packages.org.*` | jOOR（`org.joor.Reflect`）、slf4j 日志等 |
| `Packages.javax.*`、`Packages.javassist.*`、`Packages.it.unimi.dsi.fastutil.*` | Swing、字节码、fastutil 等依赖库 |

!!! note "d.ts 没声明 ≠ 运行时没有"
    d.ts 只收录 JsMacros API 引用到的类和成员。运行时的 `Packages` 是真正的包访问器，**类路径上的任何类都能访问**——比如 `Packages.java.text.SimpleDateFormat` 没出现在 d.ts 里，但照样能用，只是编辑器没有补全、可能标红。

!!! warning "为什么 Packages 里没有 net.minecraft"
    d.ts 里 Minecraft 类一律写成 `/* net.minecraft.client.Minecraft */ any` 这样的注释。因为正式客户端里 Minecraft 类是**混淆名**（如 `net.minecraft.class_310`），开发环境才叫 `net.minecraft.client.MinecraftClient`——同一个类两个名字，没法静态声明。访问 MC 内部类请走 Helper 的 `getRaw()`、`Client.getMinecraft()`，或 `Reflection.getClass` 的双名重载。

## JavaWrapper：把 JS 函数包装成 Java 回调

### 为什么回调必须包装

JS 语言规范要求**同一时刻只能有一个线程持有一个脚本上下文**。而 JsMacros 里事件是从各种 Java 线程（游戏主线程、网络线程、其他脚本线程）发起的——如果 Java 直接调用你的 JS 函数，就是跨线程闯入你的上下文，要么崩溃要么数据错乱。

`JavaWrapper.methodToJava` 系列把 JS 函数包装成 `MethodWrapper` 对象来解决这个问题：d.ts 的 JSDoc 原文说明，JS 实现用一个**非抢占式优先级队列**管理所有想调用这个包装的线程——调用方排队，等脚本上下文空闲了才轮到自己执行 JS 代码。

另外一个直接原因：`JsMacros.on` 等方法的 Java 签名要求 `MethodWrapper` 参数，传裸 JS 函数会直接抛类型错误。

### 全方法表

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `methodToJava(c)` | `MethodWrapper` | **同步**包装。调用方等待 JS 执行完并拿到返回值 |
| `methodToJavaAsync(c)` | `MethodWrapper` | **异步**包装。不需要返回值的调用（`accept`/`run`）排进队列后立即返回 |
| `methodToJavaAsync(priority, c)` | `MethodWrapper` | 同上，并指定队列优先级（JS/JEP 专用）。JSDoc 注明：有返回值的场合也可以用这个重载来调优先级 |
| `deferCurrentTask()` | `void` | 把当前任务放回队列末尾，让别的排队任务先跑（JS/JEP 专用，小心循环等待） |
| `deferCurrentTask(priorityAdjust)` | `void` | 同上，并按 `priorityAdjust` 调整优先级 |
| `getCurrentPriority()` | `number` | 当前任务的队列优先级（JS/JEP 专用） |
| `stop()` | `void` | 关闭当前脚本上下文（详见[脚本上下文](script_context.md)） |

回调统一是两个参数 `(a, b)`：事件监听里是 `(event, context)`，`forEach` 里是 `(元素)`，`Map.forEach` 里是 `(key, value)`——取决于 Java 那头把它当哪个函数式接口用。

### 同步还是异步？

两者的差别在于**Java 调用方要不要等你**：

- **`methodToJava`（同步）**：Java 调用方阻塞排队，等 JS 函数执行完、拿到返回值才继续。**必须拿返回值的场合只能用它**——比较器、过滤器、`removeIf` 谓词等。代价是：如果你的脚本上下文正忙（比如主脚本线程在 `Time.sleep` 循环里），调用方就得干等；调用方若是游戏主线程，游戏就卡住，还可能死锁。
- **`methodToJavaAsync`（异步）**：对不需要返回值的调用（事件监听、`Runnable`、`Consumer`），把任务丢进队列后**立即返回**，调用方不被脚本拖累。若被当作需要返回值的接口调用（`apply`/`test` 等），它仍不得不等待结果——所以"需要返回值"的场合异步并不能救你。

选择准则：

| 场景 | 用哪个 |
| --- | --- |
| `JsMacros.on` 事件监听 | `methodToJavaAsync`（事件线程不该被你的脚本队列卡住） |
| `list.sort(...)`、`removeIf(...)` 等要返回值 | `methodToJava` |
| `Client.runOnMainThread(...)` | 都可以，但回调必须短——它跑在游戏主线程上，有看门狗 |
| 大量高频事件（Tick 等） | `methodToJavaAsync`，必要时用 `priority` 重载分主次 |

```javascript
// 异步：标准的事件监听写法
const listener = JsMacros.on("Tick", JavaWrapper.methodToJavaAsync((e, ctx) => {
    // 注意：这里用参数 ctx，不要用全局 context
}))

// 同步：Java 排序需要比较结果，必须同步
const list = JavaUtils.createArrayList([3, 1, 2])
list.sort(JavaWrapper.methodToJava((a, b) => a - b))
Chat.log(list.toString()) // [1, 2, 3]

JsMacros.off(listener)
```

!!! warning "包装回调里的注意点"
    1. **回调跑在创建它的脚本上下文里**。上下文被关闭（脚本正常结束且没有存活监听器、或调用了 `JavaWrapper.stop()`）后，包装就失效了。注册了监听器的脚本会被 JsMacros 保活，详见[脚本上下文](script_context.md)。
    2. **回调的第二个参数才是本次事件的 `context`**，全局 `context` 属于最初那次脚本运行。
    3. **回调里可以照常调 JsMacros API**（`Chat.log`、`Player.getPlayer()` 等），但同一个上下文里的回调是排队执行的：一个回调里 `Time.sleep(5000)`，同上下文的其他回调就得等 5 秒。长耗时逻辑要么拆成独立脚本，要么接受排队，要么用 `deferCurrentTask()` 主动让位。
    4. joined 事件回调里先办正事、早点 `releaseLock()`，别在锁内 sleep——细节见[脚本上下文](script_context.md)。

## MethodWrapper：能冒充一切函数式接口的对象

`JavaWrapper.methodToJava*` 返回的 `MethodWrapper`（d.ts 47086 行，`Packages.xyz.wagyourtail.jsmacros.core.MethodWrapper<T, U, R, C>`）同时实现了 Java 常用的函数式接口：

`Runnable`、`Comparator<T>`、`Consumer<T>`、`BiConsumer<T, U>`、`Function<T, R>`、`BiFunction<T, U, R>`、`Predicate<T>`、`BiPredicate<T, U>`、`Supplier<R>`

也就是说，**任何**接受这些接口的 Java API 都能直接收下一个 `MethodWrapper`。d.ts 里 Java 集合的签名干脆直接写成了 `MethodWrapper`：

```typescript
// Packages.java.util.ArrayList 节选（d.ts 21219–21223 行）
forEach(arg0: MethodWrapper<E>): void;
removeIf(arg0: MethodWrapper<E, any, boolean>): boolean;
sort(arg0: MethodWrapper<E, E, int>): void;
```

### 方法表

| 方法 | 来自接口 | 说明 |
| --- | --- | --- |
| `accept(t)` / `accept(t, u)` | Consumer / BiConsumer | 消费一个/两个参数，无返回值 |
| `apply(t)` / `apply(t, u)` | Function / BiFunction | 计算并返回结果 |
| `test(t)` / `test(t, u)` | Predicate / BiPredicate | 返回布尔值 |
| `run()` | Runnable | 无参无返回值执行 |
| `get()` | Supplier | 无参取值 |
| `compare(a, b)` | Comparator | 返回比较用的整数 |
| `andThen(after)` | — | 串联另一个 `MethodWrapper`，前者执行完执行后者（脚本里 `after` 也必须是 `MethodWrapper`） |
| `negate()` | — | 返回谓词取反的包装 |
| `getCtx()` | — | 返回创建它的脚本上下文（`BaseScriptContext`），可能为 `null` |
| `overrideThread()` | — | 返回非空则覆盖 `JsMacros.on` 使用的线程（JEP 用，JS 脚本不用管） |
| `preventSameScriptJoin()` | — | 已弃用 |

在脚本里也可以手动调用这些方法——这就是"在 JS 里调用被包装的 JS 函数"，同样会走队列：

```javascript
const double = JavaWrapper.methodToJava((x) => x * 2)
const addOne = JavaWrapper.methodToJava((x) => x + 1)
const combined = double.andThen(addOne)

Chat.log(combined.apply(5)) // 11 ( 5*2 + 1 )

// 传给 Java 集合 API
const nums = JavaUtils.createArrayList([5, 3, 8, 1])
nums.removeIf(JavaWrapper.methodToJava((x) => x > 4))
nums.forEach(JavaWrapper.methodToJava((x) => Chat.log(`剩下: ${x}`))) // 3, 1
```

## JavaUtils：创建 Java 集合与杂项工具

JS 数组、JS 对象和 Java 的 `List`/`Map` 不是一回事。传给 JsMacros API 时 GraalJS 通常能自动转换，但直接传给**原始 Java API**（Mod 接口、反射调用、MC 内部方法）时，对方要的是真正的 `java.util.List`/`Map`——这时用 `JavaUtils` 创建：

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `createArrayList()` | `java.util.ArrayList` | 空列表 |
| `createArrayList(array)` | `java.util.ArrayList<T>` | 由 JS 数组填充 |
| `createHashMap()` | `java.util.HashMap` | 空映射 |
| `createHashSet()` | `java.util.HashSet` | 空集合 |
| `getRandom()` | `java.util.SplittableRandom` | 随机数生成器 |
| `getRandom(seed)` | `java.util.SplittableRandom` | 固定种子，每次得到相同序列 |
| `getHelperFromRaw(raw)` | 对应 Helper 或 `null` | 把原始对象包装回正确的 Helper（`getRaw()` 的逆操作） |
| `arrayToString(array)` | `string` | 数组转字符串表示 |
| `arrayDeepToString(array)` | `string` | 多维数组转字符串（内部元素也转成字符串） |

```javascript
const map = JavaUtils.createHashMap()
map.put("key", "value")
Chat.log(map.get("key"))

const rng = JavaUtils.getRandom(42)   // 固定种子, 每次运行同一序列
Chat.log(rng.nextInt(1, 101))         // [1, 100] 内的整数

// 原始对象 → Helper
const rawPlayer = Player.getPlayer().getRaw()
const helper = JavaUtils.getHelperFromRaw(rawPlayer)
Chat.log(helper.getName().getString())
```

## Reflection：反射库

`Reflection`（d.ts 2657–2845 行）覆盖了从"拿类"到"运行时编译 Java"的完整链条。

### 全方法表

**类与实例：**

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getClass(name)` | `Class` | 按全限定名取类，支持小写基本类型名（`"int"` 等） |
| `getClass(name, name2)` | `Class` | 先试 `name` 再试 `name2`——为 MC 混淆名/开发名双写设计 |
| `newInstance(c, ...objects)` | 实例 | 创建实例。JS 里一般直接 `new` 就行，这个主要给 Lua 用 |
| `getClassName(o)` | `string` | 任意对象/类的全限定类名（`.` 分隔） |

**方法与字段：**

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `getDeclaredMethod(c, name, ...parameterTypes)` | `Method` | 本类声明的方法（含私有） |
| `getDeclaredMethod(c, name, name2, ...parameterTypes)` | `Method` | 双名版本 |
| `getMethod(c, name, ...parameterTypes)` | `Method` | public 方法（含继承） |
| `getMethod(c, name, name2, ...parameterTypes)` | `Method` | 双名版本 |
| `getDeclaredField(c, name)` | `Field` | 本类声明的字段（含私有） |
| `getDeclaredField(c, name, name2)` | `Field` | 双名版本 |
| `getField(c, name)` | `Field` | public 字段（含继承） |
| `getField(c, name, name2)` | `Field` | 双名版本 |
| `invokeMethod(m, c, ...objects)` | `any` | 调用方法，数字参数自动做类型适配；静态方法 `c` 传 `null` |
| `getReflect(obj)` | `org.joor.Reflect` | jOOR 链式反射包装（见下文） |

**加载与编译：**

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `loadJarFile(file)` | `boolean` | 加载 jar 到类路径，路径**相对脚本文件夹** |
| `compileJavaClass(className, code)` | `Class` | 运行时编译 Java 类（要求装了 JDK） |
| `getCompiledJavaClass(className)` | `Class \| null` | 最近一次编译的版本 |
| `getAllCompiledJavaClassVersions(className)` | `JavaList<Class>` | 全部编译版本，按编译顺序 |
| `createLibrary(className, javaCode)` | `void` | 编译并注册一个 JsMacros 库类 |

**代理与类构建（进阶）：**

| 方法 | 返回 | 说明 |
| --- | --- | --- |
| `createClassProxyBuilder(clazz, ...interfaces)` | `ProxyBuilder<T>` | 在脚本语言里"继承"Java 类，带正确的线程支持 |
| `createClassBuilder(cName, clazz, ...interfaces)` | `ClassBuilder<T>` | 运行时构建新类 |
| `getClassFromClassBuilderResult(cName)` | `Class` | 取回构建结果 |
| `createLibraryBuilder(name, perExec, ...acceptedLangs)` | `LibraryBuilder` | 构建自定义 JsMacros 库 |

### 混淆名与双名机制

1.21.1 正式客户端里，Minecraft 类、方法、字段都是 **intermediary 混淆名**（`class_310`、`method_xxxx`、`field_xxxx`）；开发环境（Yarn/Mojmap 映射）才是可读名。**同一段反射代码在两种环境下需要的名字不一样**——这就是 `getClass`/`getMethod`/`getField` 都有 `name, name2` 双名重载的原因：先试第一个，失败再试第二个，两头都能跑。

```javascript
// MinecraftClient 的 intermediary 名是 net.minecraft.class_310
// 双名写法在正式端和开发端都能拿到
const mcClass = Reflection.getClass("net.minecraft.class_310", "net.minecraft.client.MinecraftClient")

// 看看你所在的环境用的是哪个名字
const mc = Client.getMinecraft()
Chat.log(Reflection.getClassName(mc))
// 正式客户端: net.minecraft.class_310
// 开发环境:   net.minecraft.client.MinecraftClient
```

!!! warning "反射 MC 内部成员前先查映射"
    方法/字段的 intermediary 名（`method_1551` 之类）没法猜，必须查映射表。d.ts 对 `Client.getMinecraft()` 的注释就推荐了 [Minecraft Mappings Viewer](https://wagyourtail.xyz/Projects/Minecraft%20Mappings%20Viewer/App)：选好版本后搜类名，把 intermediary 名和 yarn 名分别填进双名参数。只写一个名字的脚本换个环境（或换个 MC 版本）就会 `NoSuchMethodException`。能用 Helper 解决的事永远别用反射。

### 实战：完整的反射流程

拿标准库演示（`java.*` 没有混淆问题，示例绝对稳定）——反射调用静态方法 `Runtime.getRuntime()`，再读 JVM 内存：

```javascript
const RuntimeClass = Reflection.getClass("java.lang.Runtime")
const getRuntime = Reflection.getMethod(RuntimeClass, "getRuntime")

// 静态方法: 第二个参数传 null
const runtime = Reflection.invokeMethod(getRuntime, null)

const used = (runtime.totalMemory() - runtime.freeMemory()) / 1048576
const max = runtime.maxMemory() / 1048576
Chat.log(`JVM 内存: ${Math.round(used)} / ${Math.round(max)} MB`)
```

### jOOR：链式反射

逐个 `getDeclaredField` + `setAccessible` 太啰嗦时，用 `Reflection.getReflect(obj)` 拿 [jOOR](https://github.com/jOOQ/jOOR) 包装，私有成员也能直接链式访问：

| 方法 | 说明 |
| --- | --- |
| `field(name)` | 取字段，返回新的 `Reflect` 包装 |
| `fields()` | 所有字段，`JavaMap<string, Reflect>` |
| `get()` / `get(name)` | 解包出实际值 |
| `set(name, value)` | 写字段 |
| `call(name)` / `call(name, ...args)` | 调用方法（含私有） |
| `create(...args)` | 调构造器 |
| `type()` | 被包装对象的 `Class` |

```javascript
// 列出 Minecraft 客户端对象的所有字段名——直观感受当前环境的 mappings
const mc = Client.getMinecraft()
const names = []
Reflection.getReflect(mc).fields().keySet().forEach(
    JavaWrapper.methodToJava((name) => names.push(name))
)
Chat.log(`共 ${names.length} 个字段, 前 10 个: ${names.slice(0, 10).join(", ")}`)
// 正式客户端里看到的是 field_1687 这样的混淆名
```

## 加载 jar 与运行时编译 Java

```javascript
// 路径相对脚本文件夹
Reflection.loadJarFile("libs/example.jar")
const Example = Reflection.getClass("com.example.Example")
```

运行时编译整个类需要安装 JDK（且可能需要用 JDK 启动 Minecraft）：

```javascript
Reflection.compileJavaClass("example.Hello", `
package example;
public class Hello {
    public String hi() { return "hi"; }
}
`)

const HelloClass = Reflection.getCompiledJavaClass("example.Hello")
const hello = Reflection.newInstance(HelloClass)
Chat.log(hello.hi())
```

d.ts 的 JSDoc 强调两点：编译出的类**不能被脚本语言直接按名字访问**，要通过 `GlobalVars.putObject` 存起来或从本库取回；热更新不会影响已创建的实例——重新编译后旧实例还是旧类，`getAllCompiledJavaClassVersions` 能看到全部历史版本。

`createLibrary(className, javaCode)` 更进一步：编译一个带 `@Library` 注解、继承 `BaseLibrary`（或 `PerExecLibrary` 等）的类，注册成所有脚本可用的全局库。`createClassProxyBuilder`、`createClassBuilder`、`createLibraryBuilder` 属于写扩展库的工具，普通宏脚本用不到，需要时对照 d.ts 里 `ProxyBuilder`、`ClassBuilder`、`LibraryBuilder` 的声明使用。

!!! warning "进阶危险区"
    反射私有成员、加载 jar、运行时编译都绕过了正常的兼容性保障，MC 或 JsMacros 更新后随时可能失效；jar 里的代码拥有和 Mod 一样的权限，**只加载你信任的 jar**。

## java.* 标准库速查

脚本里最常直接使用的标准库类：

| 类 | 用途 |
| --- | --- |
| `Packages.java.util.ArrayList` / `HashMap` / `HashSet` | 集合（也可用 `JavaUtils.create*`） |
| `Packages.java.util.UUID` | 玩家/实体 UUID 解析比较 |
| `Packages.java.util.Optional` | 一些 Java API 的返回值，用 `isPresent()` / `get()` 拆 |
| `Packages.java.time.LocalDateTime` + `java.time.format.DateTimeFormatter` | 日期时间格式化（推荐） |
| `Packages.java.text.SimpleDateFormat` | 老式时间格式化（运行时可用，d.ts 未声明、无补全） |
| `Packages.java.io.File` | 文件对象。全局变量 `file` 就是当前脚本的 `File`；`FS.toRawFile(path)` 把相对路径转成 `File` |
| `Packages.java.lang.Thread` | 线程（**慎用**，见下方警告） |

格式化当前时间：

```javascript
const LocalDateTime = Packages.java.time.LocalDateTime
const DateTimeFormatter = Packages.java.time.format.DateTimeFormatter

const now = LocalDateTime.now()
const text = now.format(DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss"))
Chat.log(`当前时间: ${text}`)
```

File 对象常见来源：

```javascript
Chat.log(`当前脚本: ${file.getAbsolutePath()}`)      // 全局变量 file
const cfg = FS.toRawFile("config.json")               // 相对脚本文件夹
Chat.log(`存在: ${cfg.exists()}, 大小: ${cfg.length()} 字节`)
```

!!! warning "Thread.sleep 与手动开线程"
    - 睡眠请用 `Time.sleep(millis)` 而不是 `Packages.java.lang.Thread.sleep(...)`：两者都阻塞当前线程，但 `Time.sleep` 是 JsMacros 提供的入口，配合脚本的中断/停止机制工作；游戏内等待更应该用 `Client.waitTick()`（见[时间与工具](time_utils.md)）。
    - `new Thread(wrapper).start()` 虽然能跑（`Thread` 构造器在 d.ts 里就收 `MethodWrapper`），但这个线程**不归脚本上下文管**：脚本被停止时它不会被终止，异常也没人接。需要并行就开多个脚本/服务，或用 `methodToJavaAsync`，让 JsMacros 管理线程。

## 相关页面

- [常用类型](types.md)——JS/Java 类型互转、Helper 与 `getRaw()`、全局类型别名
- [脚本上下文](script_context.md)——线程模型、joined、上下文保活、`deferCurrentTask` 的调度背景
- [时间与工具](time_utils.md)——`Time`、`Utils`、`JavaUtils` 的日常用法
- [外部 API 总览](external_api.md) / [Baritone API](baritone_api.md)——调其他 Mod 的 Java API
