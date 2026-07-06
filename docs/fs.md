---
icon: lucide/files
---

# 文件系统

`FS` 提供文件和目录操作。路径通常相对 JsMacros 的宏目录，适合保存配置、日志、缓存和导出数据。

## 路径和判断

```javascript
Chat.log(FS.exists("config.json"))
Chat.log(FS.isFile("config.json"))
Chat.log(FS.isDir("logs"))
Chat.log(FS.getName("logs/latest.txt"))
Chat.log(FS.getDir("logs/latest.txt"))
```

| 方法 | 作用 |
| --- | --- |
| `list(path)` | 列出目录，失败返回 `null` |
| `exists(path)` | 是否存在 |
| `isDir(path)` / `isFile(path)` | 是否目录/文件 |
| `getName(path)` | 文件名 |
| `getDir(path)` | 父目录 |
| `combine(patha, pathb)` | 拼路径 |
| `toRelativePath(absolutePath)` | 转相对路径 |

## 创建、复制、移动、删除

```javascript
FS.makeDir("logs")
FS.createFile("logs", "latest.txt", true)
FS.copy("logs/latest.txt", "logs/backup.txt")
FS.move("logs/backup.txt", "logs/old.txt")
FS.unlink("logs/old.txt")
```

!!! warning "删除是真的删除"
    `FS.unlink(path)` 会删除文件或空目录。写脚本清理文件前先 `Chat.log(path)` 确认路径。

## 读写文件

```javascript
const file = FS.open("logs/latest.txt")
file.write("第一行\n")
file.append("第二行\n")
Chat.log(file.read())
file.close()
```

!!! warning "write 会覆盖"
    `FS.open(path).write(text)` 是覆盖写。追加日志请用 `append(text)`。

二进制：

```javascript
const bytes = FS.open("data.bin").readBytes()
```

指定字符集：

```javascript
const file = FS.open("legacy.txt", "GBK")
```

## 按行读取

```javascript
const lines = FS.open("logs/latest.txt").readLines()
while (lines.hasNext()) {
    Chat.log(lines.next())
}
lines.close()
```

## 遍历文件

```javascript
FS.walkFiles("logs", 2, false, JavaWrapper.methodToJava((path, attrs) => {
    Chat.log(path)
}))
```

参数：

| 参数 | 含义 |
| --- | --- |
| `path` | 起始目录 |
| `maxDepth` | 最大深度 |
| `followLinks` | 是否跟随链接 |
| `visitor` | 回调，接收路径和属性 |

## Java 原始对象

```javascript
const rawFile = FS.toRawFile("config.json")
const rawPath = FS.toRawPath("config.json")
const attrs = FS.getRawAttributes("config.json")
```

这些适合和 Java API 混用，普通脚本少用。

