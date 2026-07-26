---
icon: lucide/files
---

# 文件系统（FS）

`FS` 提供文件和目录操作，适合保存脚本配置、写日志、缓存数据和导出信息。本页覆盖 `FS` 命名空间的全部方法，以及 `FS.open()` 返回的 `FileHandler` 文件句柄。

## 相对路径从哪里算起？

d.ts 中几乎每个 `FS` 方法的 JSDoc 都写着同一句话：*relative to the script's folder*——**相对路径以"当前脚本文件所在的文件夹"为根目录**，而不是固定的 `Macros` 目录。

- 脚本直接放在 `Macros` 根目录下：两者是同一个地方，感觉不出区别。
- 脚本放在 `Macros/my_scripts/` 子文件夹里：`FS.open("config.json")` 打开的是 `Macros/my_scripts/config.json`。

!!! tip "同一个文件夹里的脚本共享配置很方便"
    把一组相关脚本放进同一个子文件夹，它们用相同的相对路径就能读到同一个配置文件，不用写绝对路径。

### 全局变量 `file`

每个脚本运行时都自带一个全局变量 `file`，它是当前脚本文件对应的 Java `File` 对象（`java.io.File`）：

```javascript
Chat.log(file.getName())          // 脚本文件名，如 "test.js"
Chat.log(file.getParent())        // 脚本所在文件夹的路径
Chat.log(file.getAbsolutePath())  // 脚本的绝对路径
```

!!! warning "不要覆盖全局 file"
    写 `const file = FS.open(...)` 会遮住全局变量 `file`。本页示例统一用 `fh`（file handler）作为变量名。

## 判断与路径工具

```javascript
Chat.log(FS.exists("config.json"))       // 是否存在
Chat.log(FS.isFile("config.json"))       // 是不是文件
Chat.log(FS.isDir("logs"))               // 是不是目录

// 列出目录内容，返回文件名数组；失败（如目录不存在）返回 null
const files = FS.list(".")
if (files) {
    for (const name of files) {
        Chat.log(name)
    }
}
```

| 方法 | 返回值 | 作用 |
| --- | --- | --- |
| `list(path)` | `JavaArray<string> \| null` | 列出目录中的文件名，失败返回 `null` |
| `exists(path)` | `boolean` | 路径是否存在 |
| `isDir(path)` | `boolean` | 是否为目录 |
| `isFile(path)` | `boolean` | 是否为文件 |
| `getName(path)` | `string` | 取路径最后一段（文件名） |
| `getDir(path)` | `string` | 取文件的目录部分（目录则取父目录） |
| `combine(patha, pathb)` | `string` | 拼接两段路径 |
| `toRelativePath(absolutePath)` | `string` | 把绝对路径转成相对脚本文件夹的路径 |

## 创建、复制、移动、删除

| 方法 | 返回值 | 作用 |
| --- | --- | --- |
| `makeDir(path)` | `boolean` | 创建目录，返回是否成功 |
| `createFile(path, name)` | `boolean` | 在 `path` 下新建文件 `name`，**父目录必须已存在** |
| `createFile(path, name, createDirs)` | `boolean` | 同上，`createDirs` 为 `true` 时自动创建缺失的父目录 |
| `copy(from, to)` | `void` | 复制文件（出错抛 `IOException`） |
| `move(from, to)` | `void` | 移动/重命名文件（出错抛 `IOException`） |
| `unlink(path)` | `boolean` | 删除文件，返回是否成功 |

```javascript
FS.makeDir("logs")
FS.createFile("logs", "latest.txt", true)   // true: 自动补建父目录
FS.copy("logs/latest.txt", "logs/backup.txt")
FS.move("logs/backup.txt", "logs/old.txt")
FS.unlink("logs/old.txt")
```

!!! warning "删除是真的删除"
    `FS.unlink(path)` 直接删除文件，不进回收站。写清理脚本前先 `Chat.log(path)` 确认路径无误。

## FS.open 与 FileHandler

```typescript
FS.open(path: string): FileHandler
FS.open(path: string, charset: string): FileHandler
```

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `path` | `string` | 文件路径，相对脚本所在文件夹 |
| `charset` | `string` | 可选，读写用的字符集，默认 `UTF-8` |

返回一个 `FileHandler` 文件句柄，之后的读写都通过它进行：

```javascript
const fh = FS.open("logs/latest.txt")
fh.write("第一行\n")        // 覆盖写入
fh.append("第二行\n")       // 追加
Chat.log(fh.read())         // 读出全部内容
```

读取老编码的文件时指定字符集：

```javascript
const fh = FS.open("legacy.txt", "GBK")
```

!!! note "FileHandler 不需要手动关闭"
    `FileHandler` 本身没有 `close()` 方法——`read`/`write`/`append` 每次调用都是独立完成的。只有 `readLines()` 和 `streamBytes()` 返回的迭代器/流需要你手动 `close()`。

### 写入与追加

| 方法 | 返回值 | 作用 |
| --- | --- | --- |
| `write(s)` | `this` | 写入字符串，**覆盖整个文件** |
| `write(b)` | `this` | 写入字节数组（`byte[]`），同样是覆盖 |
| `append(s)` | `this` | 在文件末尾追加字符串 |
| `append(b)` | `this` | 在文件末尾追加字节数组 |

!!! warning "write 会覆盖"
    `write()` 是破坏性操作，会替换文件的全部内容。记日志请用 `append()`。

这四个方法都返回 `FileHandler` 自身，可以链式调用：

```javascript
FS.open("logs/latest.txt")
    .write("=== 日志开始 ===\n")
    .append("第一条记录\n")
    .append("第二条记录\n")
```

### 读取

| 方法 | 返回值 | 作用 |
| --- | --- | --- |
| `read()` | `string` | 读出整个文件为字符串 |
| `readBytes()` | `JavaArray<number>` | 读出整个文件为字节数组 |
| `readLines()` | `FileLineIterator` | 按行迭代读取（见下） |
| `streamBytes()` | `java.io.BufferedInputStream` | 获取输入流，用完记得 `close()` |
| `getFile()` | `java.io.File` | 取底层的 Java `File` 对象 |

### 按行读取大文件

`readLines()` 返回一个行迭代器 `FileLineIterator`，只有 `hasNext()`、`next()`、`close()` 三个方法。它不会把整个文件读进内存，适合大文件；**用完必须 `close()`**，否则会泄漏文件句柄：

```javascript
const lines = FS.open("logs/latest.txt").readLines()
try {
    while (lines.hasNext()) {
        Chat.log(lines.next())
    }
} finally {
    lines.close()
}
```

## 读写 JSON 配置文件（推荐模式）

保存脚本配置最常见的做法：写入时 `JSON.stringify` + `write`，启动时 `read` + `JSON.parse` 并用 `try/catch` 容错。下面是一个可以直接套用的完整模板：

```javascript
const CONFIG_PATH = "my_script_config.json"

// 默认配置：文件不存在或损坏时的兜底
const defaults = {
    enabled: true,
    radius: 5,
    whitelist: ["Steve", "Alex"]
}

function loadConfig() {
    if (!FS.exists(CONFIG_PATH)) return Object.assign({}, defaults)
    try {
        const text = FS.open(CONFIG_PATH).read()
        // 和默认值合并：以后新增配置项，旧配置文件也不会缺字段
        return Object.assign({}, defaults, JSON.parse(text))
    } catch (e) {
        Chat.log("§c配置读取失败，已使用默认值: " + e)
        return Object.assign({}, defaults)
    }
}

function saveConfig(config) {
    // null, 4 表示缩进 4 格，方便手动编辑
    FS.open(CONFIG_PATH).write(JSON.stringify(config, null, 4))
}

// 使用
const config = loadConfig()
Chat.log("当前半径: " + config.radius)
config.radius = 8
saveConfig(config)
```

!!! tip "为什么要 try/catch"
    配置文件可能被手动改坏（少个逗号、编码错误），`JSON.parse` 会直接抛异常。有了 `try/catch` 兜底，脚本永远能带着默认配置正常启动，而不是一启动就报错。

## 遍历目录树：walkFiles

```typescript
FS.walkFiles(path: string, maxDepth: int, followLinks: boolean, visitor): void
```

| 参数 | 类型 | 说明 |
| --- | --- | --- |
| `path` | `string` | 起始目录（相对脚本文件夹） |
| `maxDepth` | `int` | 最大递归深度，太大可能导致栈溢出 |
| `followLinks` | `boolean` | 是否跟随符号链接 |
| `visitor` | `MethodWrapper` | 对每个文件调用，参数是文件路径和文件属性 |

```javascript
FS.walkFiles("logs", 2, false, JavaWrapper.methodToJava((path, attrs) => {
    Chat.log(path + "  " + attrs.size() + " 字节")
}))
```

回调的第二个参数是 `java.nio.file.attribute.BasicFileAttributes`，常用 `attrs.size()`（大小）、`attrs.isDirectory()`（是否目录）、`attrs.lastModifiedTime()`（修改时间）。

## Java 原始对象

需要和 Java API 混用时才用得到，普通脚本很少碰：

| 方法 | 返回值 | 作用 |
| --- | --- | --- |
| `toRawFile(path)` | `java.io.File` | 取相对路径对应的 `File` 对象 |
| `toRawPath(path)` | `java.nio.file.Path` | 取相对路径对应的 `Path` 对象 |
| `getRawAttributes(path)` | `BasicFileAttributes` | 取文件属性（大小、修改时间等） |

```javascript
const rawFile = FS.toRawFile("config.json")
const rawPath = FS.toRawPath("config.json")
const attrs = FS.getRawAttributes("config.json")
Chat.log("大小: " + attrs.size() + " 字节")
```
