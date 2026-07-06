---
icon: lucide/puzzle
---

# 语言扩展

JsMacros 默认常用 JavaScript / TypeScript 工作流，也支持通过语言扩展运行其他语言。扩展 jar 放在：

```text
.minecraft/config/jsMacros/LanguageExtensions/
```

## 什么时候需要扩展？

| 场景 | 建议 |
| --- | --- |
| 普通宏、HUD、背包脚本 | 先用 JavaScript |
| 需要类型提示和构建流程 | 用 TypeScript 模板编译到 JS |
| 已有 Python/Lua/Ruby/Kotlin 代码 | 再考虑语言扩展 |
| 追求最高兼容性 | JS 最稳 |

## 安装步骤

1. 下载对应 JsMacros 版本的语言扩展 jar。
2. 放到 `LanguageExtensions/` 目录。
3. 重启 Minecraft。
4. 在 JsMacros GUI 里选择对应脚本语言。

!!! warning "版本要对上"
    语言扩展和 JsMacros 主 mod 版本不匹配时，可能直接加载失败或运行时报错。

## 和 JavaScript 的关系

全局库仍然是同一批概念：`Chat`、`Client`、`Player`、`World`、`JsMacros` 等。只是语法和 Java 互操作方式会变。

中文文档会优先覆盖 JavaScript 写法。其他语言可以按同一 API 思路迁移。

