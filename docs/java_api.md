---
icon: lucide/code-2
---

# Java 与反射

JsMacros 脚本运行在 JVM 环境里，可以访问 Java 类、Minecraft 原始对象和反射 API。普通脚本优先用 helper；只有 helper 不够时再进入这一层。

如果目标是调用其他 Mod 暴露的 Java API，比如 Baritone，先看 [外部 API 总览](external_api.md)，再看 [Baritone API](baritone_api.md)。

## JavaWrapper

最常见的 Java 互操作不是反射，而是把 JS 函数包装成 Java 回调。

```javascript
JsMacros.on("Tick", JavaWrapper.methodToJava((e, context) => {
    Chat.log("tick")
}))
```

| 方法 | 作用 |
| --- | --- |
| `methodToJava(fn)` | 同步回调 |
| `methodToJavaAsync(fn)` | 异步回调 |
| `methodToJavaAsync(priority, fn)` | 指定优先级 |
| `stop()` | 停止当前脚本 |

事件、文件遍历、世界遍历、异步请求回调都会用到它。

## JavaUtils

```javascript
const list = JavaUtils.createArrayList()
const map = JavaUtils.createHashMap()
const set = JavaUtils.createHashSet()
```

如果 Java 方法要求 `java.util.List` 或 `java.util.Map`，用这些方法创建会比 JS 数组更稳。

## Packages 和原始对象

类型文件里可以看到大量 `Packages.xxx`。这代表 Java 包路径。

```javascript
const File = Packages.java.io.File
const f = new File("test.txt")
Chat.log(f.getAbsolutePath())
```

helper 经常提供 `getRaw...()`：

```javascript
const block = World.getBlock(0, 64, 0)
const rawState = block.getRawBlockState()
```

!!! tip "优先用 helper"
    helper 已经把很多 Minecraft 内部差异包装好了。直接操作 raw 对象时，版本更新更容易坏。

## Reflection 基础

```javascript
const clazz = Reflection.getClass("java.lang.String")
const method = Reflection.getMethod(clazz, "length")
const value = Reflection.invokeMethod(method, "hello")
Chat.log(value)
```

常用方法：

| 方法 | 作用 |
| --- | --- |
| `getClass(name)` | 获取 Java 类 |
| `getDeclaredMethod(c, name, ...types)` | 获取声明方法 |
| `getMethod(c, name, ...types)` | 获取 public 方法 |
| `getDeclaredField(c, name)` | 获取声明字段 |
| `getField(c, name)` | 获取 public 字段 |
| `invokeMethod(m, obj, ...args)` | 调用方法 |
| `newInstance(c, ...args)` | 创建实例 |
| `getReflect(obj)` | jOOR Reflect 包装 |
| `getClassName(o)` | 获取完整类名 |

## 加载 jar 和编译 Java

```javascript
Reflection.loadJarFile("libs/example.jar")
```

运行时编译：

```javascript
const clazz = Reflection.compileJavaClass("example.Hello", `
package example;
public class Hello {
    public String hi() { return "hi"; }
}
`)
```

查询编译结果：

```javascript
const clazz = Reflection.getCompiledJavaClass("example.Hello")
const versions = Reflection.getAllCompiledJavaClassVersions("example.Hello")
```

!!! warning "进阶危险区"
    反射、加载 jar、运行时编译 Java 都可能破坏兼容性，也可能带来安全风险。只加载自己信任的 jar。

## 代理类和库

`Reflection` 还提供：

| 方法 | 作用 |
| --- | --- |
| `createClassProxyBuilder(clazz, ...interfaces)` | 创建代理类构建器 |
| `createClassBuilder(cName, clazz, ...interfaces)` | 创建类构建器 |
| `createLibraryBuilder(name, perExec, ...acceptedLangs)` | 创建 JsMacros 库 |
| `createLibrary(className, javaCode)` | 编译并注册库 |

这些适合写扩展库，不是普通宏脚本的必需知识。
