# JsMacros 中文文档

面向中文玩家和脚本作者的**非官方** [JsMacros](https://modrinth.com/mod/jsmacros) 文档。JsMacros 是一个 Minecraft 客户端模组，让你用 JavaScript 编写宏脚本——自动钓鱼、聊天过滤、HUD 绘制、容器操作、自定义命令等等。

> 在线阅读：<http://jsmacros.mynotes.world/>

## 这份文档有什么不一样

- **中文 + 可运行示例**：每个方法表都配能直接粘进游戏跑的示例，而不只是签名罗列。
- **覆盖完整**：59 个事件、65 个特化实体 Helper、17 种容器子类、全部全局库，都有专门的参考页。
- **踩坑提示**：看门狗、`joined` 阻塞、监听器残留、反作弊发包顺序等实战问题单独标注。

> [!NOTE]
> 本文档非官方维护。官方文档在 <https://jsmacros.wagyourtail.xyz/>，可配合阅读；两者冲突时**以本仓库的 `JsMacros-2.1.0.d.ts` 为准**。

## 快速开始

不熟悉 JsMacros 的话，按这个顺序读：

1. [快速开始](docs/quick_start.md) — 安装模组、目录结构、跑通第一个脚本
2. [脚本模板](docs/script_template.md) — 开关脚本、安全睡眠、监听器清理的标准写法
3. [API 总览](docs/api_overview.md) — 全局库地图 + "我想做 X 该看哪页"索引
4. [事件系统](docs/events.md) — 监听聊天、按键、Tick、容器

## 页面地图

<details>
<summary>脚本基础</summary>

| 页面 | 内容 |
| --- | --- |
| [快速开始](docs/quick_start.md) | 安装、开发环境、运行方式、fork/joined、看门狗 |
| [API 总览](docs/api_overview.md) | 16 个全局库的职责地图与任务索引 |
| [全局库参考](docs/global_api_reference.md) | 全局库方法速查表 |
| [脚本模板](docs/script_template.md) | 通用模板逐段讲解 + 监听器型/服务型变体 |
| [脚本上下文](docs/script_context.md) | 线程模型、`context`/`event`/`file`、僵尸脚本原理 |
| [事件系统](docs/events.md) | `on`/`once`/`off`/`waitForEvent`、joined、自定义事件 |
| [事件参考](docs/events_reference.md) | 全部 59 个事件的字段与可取消性 |
| [事件过滤器](docs/event_filters.md) | `eventFilters()` 工厂与 Java 侧过滤 |
| [服务脚本](docs/services.md) | 常驻脚本的生命周期与规范模板 |
| [实战模式](docs/practical_patterns.md) | 13 个常用代码模式（问题 → 代码 → 讲解） |
| [常用类型](docs/types.md) | Helper 包装哲学、JS/Java 类型互转 |
| [语言扩展](docs/extensions.md) | 使用 JS 以外的语言 |

</details>

<details>
<summary>游戏 API</summary>

| 页面 | 内容 |
| --- | --- |
| [聊天与文本](docs/chat.md) | 输出/发送、§ 颜色码、TextHelper、TextBuilder、聊天历史 |
| [自定义命令](docs/commands.md) | 注册客户端命令、33 种参数类型、Tab 补全 |
| [客户端](docs/client.md) | 游戏实例、tick 调度、连接服务器、截图 |
| [游戏设置](docs/options.md) | OptionsHelper 六大设置分类读写 |
| [玩家](docs/player.md) | 玩家对象、射线检测、移动输入、统计信息 |
| [交互管理器](docs/interaction.md) | 2.x 推荐的攻击/交互/挖掘统一入口 |
| [背包](docs/inventory.md) | 槽位分区、点击模式、17 种容器子类 |
| [物品与 NBT](docs/items.md) | ItemStackHelper、附魔、NBT 解析、按 Lore 匹配 |
| [世界与方块](docs/world.md) | 世界状态、实体获取、记分板、Boss 栏、Tab 列表 |
| [世界扫描器](docs/world_scanner.md) | WorldScanner 链式过滤与异步扫描 |
| [方块 Helper](docs/blocks.md) | BlockData/Block/BlockState/BlockPos 四层概念 |
| [实体](docs/entities.md) | EntityHelper 继承链、类型转换、状态效果、村民交易 |
| [实体参考](docs/entities_reference.md) | 65 个特化实体 Helper 全方法表 |
| [位置与向量](docs/position.md) | Pos/Vec/BlockPos 互转与实用几何 |
| [键盘输入](docs/keybind.md) | 按键状态与模拟按键 |
| [HUD 渲染](docs/hud.md) | Draw2D/Draw3D/Surface、全部渲染元素 |
| [自定义图像](docs/custom_image.md) | CustomImage 内存位图绘制与纹理注册 |
| [脚本屏幕](docs/screen.md) | ScriptScreen 自定义 GUI 与控件 |

</details>

<details>
<summary>系统 API 与外部集成</summary>

| 页面 | 内容 |
| --- | --- |
| [文件系统](docs/fs.md) | 读写文件、JSON 配置、目录遍历 |
| [网络请求](docs/network.md) | HTTP 请求与 WebSocket |
| [数据包](docs/packets.md) | RecvPacket/SendPacket 与包体读写（高风险） |
| [全局变量](docs/globals.md) | 跨脚本共享状态、`toggleBoolean` 开关模式 |
| [Java 与反射](docs/java_api.md) | GraalJS 互操作、JavaWrapper、Reflection |
| [时间与工具](docs/time_utils.md) | 睡眠与 tick、哈希编码、剪贴板 |
| [外部 API 总览](docs/external_api.md) | 调用其他模组 |
| [Baritone API](docs/baritone_api.md) | 自动寻路集成 |

</details>

## 本地开发

文档用 [Zensical](https://zensical.org/)（Python）构建，不需要 Node.js。

```bash
pip install zensical

zensical serve          # 本地预览，带热重载
zensical build --clean  # 与 CI 等价的静态构建检查
```

推送到 `master`/`main` 后由 `.github/workflows/docs.yml` 自动构建并部署到 GitHub Pages。

## 目录结构

```
.
├── docs/                    # 所有文档页面（Markdown）
├── zensical.toml            # 站点配置与导航
├── JsMacros-2.1.0.d.ts      # API 事实来源，改文档前请对照它
├── .github/workflows/       # GitHub Pages CI
└── site/                    # 构建产物，不提交
```

## 参与贡献

欢迎 PR。提交前请注意：

1. **对照类型文件**：新增或修改 API 描述前，先在 `JsMacros-2.1.0.d.ts` 里确认方法名与签名，不要凭印象写。
2. **新增页面**：文件放 `docs/`，命名用小写加下划线（如 `quick_start.md`），并在 `zensical.toml` 的 `nav` 里登记。
3. **构建自检**：跑 `zensical build --clean` 确认无报错，再 `zensical serve` 检查链接、折叠块、代码块渲染是否正常。
4. **示例要能跑**：贴出的代码尽量在游戏里实测过；事件回调记得用 `JavaWrapper.methodToJava(...)` 包装。
5. **编码**：Markdown 和 TOML 一律 UTF-8。
6. **提交信息**：沿用中文短句，如 `更新<页面>: <变更点>` 或 `修复: <问题>`。

更详细的约定见 [AGENTS.md](AGENTS.md)。

## 免责声明

JsMacros 能做到很多原版客户端做不到的事。**进入多人服务器前请先确认服务器规则**，避免使用明显超出正常操作能力的脚本——文档里标注反作弊风险的地方尤其注意。因使用本文档示例导致的封号等后果，与文档作者无关。
