---
icon: lucide/home
---
# 首页

这里是 JsMacros 的中文非官方文档。它面向刚开始写 Minecraft 客户端脚本的玩家，也给已经会写脚本的人留一份能快速查 API 的中文手册。

!!! quote "说明"
    API 名称和签名以仓库里的 `JsMacros-2.1.0.d.ts` 为准。官方文档地址是 https://jsmacros.wagyourtail.xyz/ ，可以配合阅读，但遇到冲突时优先看本仓库类型文件。

## 先读哪几页？

| 目标 | 推荐页面 |
| --- | --- |
| 第一次安装和运行脚本 | [快速开始](quick_start.md) |
| 不知道有哪些全局 API | [API 总览](api_overview.md) |
| 想写按键、聊天、Tick 监听 | [事件系统](events.md) |
| 想操作玩家、背包、世界、实体 | [玩家](player.md)、[背包](inventory.md)、[世界与方块](world.md)、[实体](entities.md) |
| 想画 HUD 或 3D 标记 | [HUD 渲染](hud.md) |
| 想读写文件、请求接口、跨脚本共享状态 | [文件系统](fs.md)、[网络请求](network.md)、[全局变量](globals.md) |

## 如何参与文档贡献？

使用[GitHub](https://github.com/TheStoneFish/JsMacros-doc)提交PR即可参与文档贡献, 具体操作如下:

1. 首先Fork本仓库到自己的github账号下
2. 在自己的仓库中clone到本地
3. 在本地修改文档内容
4. 在本地提交修改

使用zensical serve进行实时预览。
## 当前文档进度

- [x] 主页
- [x] 快速开始
- [x] API 总览
- [x] 事件系统
- [x] 玩家
- [x] 背包
- [x] 世界与方块
- [x] 实体
- [x] HUD 渲染
- [x] 系统 API
- [x] Java API
- [x] 源码分析

!!! warning "反作弊提醒"
    JsMacros 可以做很多原版客户端做不到的事情。进入多人服务器前，请先确认服务器规则，并避免使用明显超出正常操作能力的脚本。
