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
| 不知道脚本该怎么组织 | [脚本模板](script_template.md)、[API 总览](api_overview.md) |
| 想写按键、聊天、Tick 监听 | [事件系统](events.md)、[事件参考](events_reference.md) |
| 想操作玩家、背包、世界、实体 | [玩家](player.md)、[背包](inventory.md)、[物品与 NBT](items.md)、[世界与方块](world.md)、[实体](entities.md) |
| 想画 HUD、3D 标记或自定义界面 | [HUD 渲染](hud.md)、[自定义图像](custom_image.md)、[脚本屏幕](screen.md) |
| 想读写文件、请求接口、跨脚本共享状态 | [文件系统](fs.md)、[网络请求](network.md)、[全局变量](globals.md) |
| 想搞懂脚本线程、监听器清理、常驻服务 | [脚本上下文](script_context.md)、[服务脚本](services.md) |
| 想监听或修改网络数据包 | [数据包](packets.md) |
| 想调用外部接口或其他 Mod | [外部 API 总览](external_api.md)、[Baritone API](baritone_api.md) |

## 如何参与文档贡献？

使用[GitHub](https://github.com/TheStoneFish/JsMacros-doc)提交 PR 即可参与文档贡献，具体操作如下：

1. 首先Fork本仓库到自己的github账号下
2. 在自己的仓库中clone到本地
3. 在本地修改文档内容
4. 在本地提交修改

使用zensical serve进行实时预览。
## 当前文档进度

- [x] 脚本基础：快速开始、API 总览、脚本模板、脚本上下文、事件系统、事件参考、事件过滤器、服务脚本、实战模式、常用类型
- [x] 游戏 API：聊天与文本、自定义命令、客户端、游戏设置、玩家、交互管理器、背包、物品与 NBT、世界与方块、世界扫描器、方块 Helper、实体、实体参考、位置与向量、键盘输入、HUD 渲染、自定义图像、脚本屏幕
- [x] 系统 API：文件系统、网络请求、数据包、全局变量、Java 与反射、时间与工具
- [x] 外部集成：外部 API 总览、Baritone API

!!! warning "反作弊提醒"
    JsMacros 可以做很多原版客户端做不到的事情。进入多人服务器前，请先确认服务器规则，并避免使用明显超出正常操作能力的脚本。
