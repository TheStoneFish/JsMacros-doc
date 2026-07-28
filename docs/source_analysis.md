---
icon: lucide/search-code
---

# 源码分析

本仓库当前主要根据 `JsMacros-2.1.0.d.ts` 写中文 API 文档。读类型文件时，可以按这条路线理解 JsMacros 的结构。

## 类型文件怎么读？

`JsMacros-2.1.0.d.ts` 大体分为几块：

| 区域 | 内容 |
| --- | --- |
| 顶部 `Events` namespace | 所有事件类型 |
| 顶层 `declare namespace` | 全局库：`Chat`、`Player`、`World` 等 |
| `Packages` namespace | Java / Minecraft / JsMacros 内部类 |
| 底部 `type` | ID、槽位、方向、屏幕名等联合类型 |

## 查 API 的方法

在仓库根目录用 `rg`：

```powershell
rg -n "declare namespace Player|function openInventory|class Inventory" JsMacros-2.1.0.d.ts
```

查方法是否弃用：

```powershell
rg -n "@deprecated|interactBlock|attack\\(" JsMacros-2.1.0.d.ts
```

查事件字段：

```powershell
rg -n "interface RecvMessage|interface SendMessage" JsMacros-2.1.0.d.ts
```

## 文档编写原则

- API 名称和签名以 `JsMacros-2.1.0.d.ts` 为准。
- 官方文档可参考结构，但不要照搬旧版本签名。
- 先写常用任务，再补完整表格。
- 示例要能运行，并尽量判空。
- 标出 deprecated、可能不同步、可能触发反作弊的地方。

## 2.1.0 重点

- `JsMacros.eventFilters()` 事件过滤器。
- `Player.getInteractionManager()` / `Player.interactions()` 作为交互主入口。
- `Draw2D.addWorldPosWrapped(...)` 和 `Draw3D.addEntityFollower(...)`。
- `EntityHelper.isReallyAlive()`、`LivingEntityHelper.isReallyAliveAndHealthy()`。
- `PlayerListEntryHelper.getCapeUrl()` / `getElytraUrl()`。
- `Request.Response.json()` 标为弃用，建议 `JSON.parse(response.text())`。

