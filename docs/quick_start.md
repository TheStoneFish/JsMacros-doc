---
icon: lucide/rocket
---

# 快速开始

这一页带你从零跑起第一个宏：安装模组、认识目录、搭建（可选的）开发环境、写第一个脚本、了解两种运行方式，以及脚本报错以后去哪里找原因。

## 获取 JsMacros

!!! warning "1.21.1 及以上版本注意"
    Modrinth 上的官方构建可能落后于最新 Minecraft 版本。对于 >= 1.21.1，可以从 GitHub 构建源码；如果游戏版本没有重大变化，也可以尝试修改 jar 内的 `fabric.mod.json` 版本限制；或者把仓库 fork 下来用 GitHub Action 构建。

从 [Modrinth](https://modrinth.com/mod/jsmacros) 获取 [JsMacros](https://modrinth.com/mod/jsmacros)，像普通 Fabric 模组一样放进 `mods` 文件夹即可。**。

## 目录结构

第一次带着 JsMacros 启动游戏后，会生成这样的目录：

```
.minecraft/
├── config/
  └── jsMacros/
    └── LanguageExtensions # 语言扩展, 如果需要使用其他语言, 请将其放入此目录
    └── Macros/ # 存放宏文件
    └── options.json # JsMacros配置
    └── options.json.v2.bak # JsMacros配置备份
```

你写的所有脚本都放在 `Macros` 目录（或它的子目录）下。

## 创建开发环境

!!! note "提示"
    非必须，但强烈建议。配置好之后写脚本会有类型提示和代码补全，能在写的时候就发现拼错的方法名，而不是等运行报错。

### 安装 VSCode

下载并安装 [VSCode](https://code.visualstudio.com/)，推荐以下插件：

- Prettier（代码格式化）
- TabOut（代码缩进）
- indent-rainbow（代码缩进高亮）

### 使用 ts 进行类型检查和代码提示

``` cmd title="cmd"
cd Macros
git clone https://github.com/TheStoneFish/JsMacros-template.git .
npm install
code .
```

克隆后使用 VSCode 打开并安装依赖，即可看到类型提示和代码补全等功能。输入 `Chat.` 时弹出的方法列表，就来自 `JsMacros-2.1.0.d.ts` 类型定义文件。

![](https://img.mynotes.world//202602262111399.png)

## 创建第一个宏

=== "js"
    ``` js title="test.js"
    Chat.log("Hello, world!")
    ```
    把文件放入 `.minecraft/config/jsMacros/Macros/src` 目录下（不用模板环境的话，直接放 `Macros` 下任意位置也可以）。
=== "ts"
    ``` ts title="test.ts"
    Chat.log("Hello, world!")
    ```
    把文件放入 `.minecraft/config/jsMacros/Macros/src` 目录下。

    使用 ts 需要先编译成 js，请提前安装依赖：
    ``` cmd title="cmd"
    npm install
    npm run build
    ```
    编译后的文件会生成到 `.minecraft/config/jsMacros/Macros/dist` 目录下，游戏里绑定的是编译后的 js 文件。

`Chat.log()` 只在你自己的聊天栏输出内容，不会发送给服务器，是最常用的调试手段。

## 运行宏

运行宏有两种方式：

| 方式 | 描述 |
| ----------- | ------------------------------------ |
| 按键 | 按下绑定的按键即可运行，同一个脚本可以同时运行多份 |
| 服务 | 在服务（Services）里运行脚本，同一个服务同时只有一份在跑，可以设置随客户端启动自动运行 |

日常测试用按键最方便；需要"常驻后台、开游戏就自动跑"的脚本用服务更合适（详见[服务](services.md)）。

!!! warning "注意"
    正在运行的脚本不会因为你退出服务器/单人存档而停止，只有退出游戏（或手动停止）才会结束。长期运行的脚本请在逻辑里自己判断 `World.isWorldLoaded()`。

!!! tip "下一步"
    会写 `Chat.log("Hello, world!")` 之后，建议立刻看[脚本模板](script_template.md)。那里会讲开关脚本、`safeSleep`、监听器清理，以及为什么脚本关闭后旧线程可能还在跑。

### 按键触发

打开 JsMacros 界面（默认按 K 键），在 KeyBind 页面新建一行，绑定按键并选择脚本文件：

![](https://img.mynotes.world//202602251554510.png)

按下对应按键即可运行。运行中的脚本会显示在屏幕左下角"正在运行"列表里，点击即可结束；同一个脚本可以同时运行多份。

图中的 fork 和 joined 是脚本的运行状态，具体含义如下：

| 状态 | 描述 |
| ----------- | ------------------------------------ |
| ![](https://img.mynotes.world//202602262322605.png) | `fork` 状态：脚本在自己的线程里运行，不会阻塞触发它的线程 |
| ![](https://img.mynotes.world//202602262323456.png) | `joined` 状态：脚本会阻塞触发它的线程，直到脚本结束或释放锁 |

默认情况下，按键触发的宏都是 `fork` 状态。因为按键触发发生在主线程上，如果改成 `joined`，脚本运行超过 500ms 就会触发看门狗被强制结束，所以按键宏正常保持 `fork` 就行。

!!! tip "看门狗"
    当宏阻塞主线程超过 500ms 时，会触发看门狗（watchdog），强制结束宏的运行，防止脚本把游戏卡死。

事件监听时同样可以设置 joined（`JsMacros.on` 的第二个参数传 `true`）。对 `Tick` 这类事件阻塞没有任何意义；但对实现了 `Cancellable` 的事件，比如：

- SendMessage
- RecvMessage
- Key
- SignEdit
- ClickSlot
- DropSlot

joined 监听可以在事件生效前修改/取消它（比如拦截 `SendMessage` 要发送的消息、取消 `RecvMessage` 收到的消息）。处理完毕后用 `context.releaseLock()` 解除阻塞——阻塞超过 500ms 同样会触发看门狗。

``` js
JsMacros.on('SendMessage', true, JavaWrapper.methodToJava((e) => {
    Chat.log(`发送消息: ${e.message}`)
    e.cancel()
    Time.sleep(1000)
}))
```

上面这段会触发看门狗，因为 joined 回调把主线程阻塞了整整 1 秒。正确写法是完成 `cancel()` 后先解除线程锁，再做耗时的事：

``` js
JsMacros.on('SendMessage', true, JavaWrapper.methodToJava((e, context) => {
    Chat.log(`发送消息: ${e.message}`)
    e.cancel()
    context.releaseLock()
    Time.sleep(1000)
}))
```

事件系统的完整讲解见[事件系统](events.md)。

按键触发的宏还可以设置触发条件，共有三种：

| 图标 | 描述 |
| ----------- | ------------------------------------ |
| ![](https://img.mynotes.world//202602262324309.png) | 松开按键触发 |
| ![](https://img.mynotes.world//202602262324360.png) | 按下按键触发 |
| ![](https://img.mynotes.world//202602262326874.png) | 松开和按下都会触发 |

### 服务触发

![](https://img.mynotes.world//202602262137967.png)

服务（Services）里的脚本可以设置随客户端启动自动运行，适合常驻型脚本。运行后需要在服务界面里关闭运行状态才会停止，同一个服务不能同时运行多份。详细用法见[服务](services.md)。

## 脚本报错去哪看

写脚本一定会遇到报错，常见的排查位置：

1. **聊天栏**：脚本抛出未捕获的异常时，JsMacros 通常会把错误信息以红色文字输出到聊天栏，把鼠标悬停在错误上一般能看到更多细节（文件名、行号）。这是最快的排查入口。
2. **游戏日志**：更完整的堆栈信息会写进游戏日志 `.minecraft/logs/latest.log`，用脚本文件名搜索即可定位。启动器（如 HMCL）的日志窗口里也能看到。
3. **服务界面**：服务脚本出错停止后，服务界面里的运行状态会发生变化，可以据此发现"服务悄悄挂了"。

!!! tip "主动打日志"
    与其等报错，不如在关键位置用 `Chat.log()` 输出中间状态；对可能出错的段落用 `try { ... } catch (e) { Chat.log(`§c出错: ${e}`) }` 包起来，能把问题定位得更准。[脚本模板](script_template.md)里有一个现成的日志分级函数。
