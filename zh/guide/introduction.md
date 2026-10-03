# 项目介绍

**OhMyLoader（OML）** 是一个面向 Minecraft Java Edition 的模组加载器：自研字节码注入引擎、统一的高层
mod API、数据驱动的声明式内容注册，客户端与专用服务端双端支持。当前稳定主线为 26.3，全线运行在 Java 27 上。

## 设计理念

### 统一语义 API

mod 面向统一的高层游戏语义编程（事件、生命周期、内容注册），不直接接触版本内部结构。跨版本的底层代差只发生在
语义层之下的驱动层（adapter），不向上泄漏。

### 自研字节码引擎

内核自带轻量级字节码手术刀，替代繁重的 SpongePowered Mixin 与 Access Transformer 底层：

- **注入 DSL**：`atHead` / `atReturn` / `beforeCall` 等锚点 + `call` / `modifyArg` / `redirectCall` 等
  载荷，语义对齐 Mixin，但完全静态声明、启动期自检；
- **Mixin 类合并**：`@Shadow` / `@Overwrite` / `@Unique` / `@Accessor` / `@Invoker` 全支持；
- **注解前端**：习惯 Mixin 的开发者可以直接写 `@Mixin` / `@Inject`，加载器编译成注入规则后走同一条管线。

每条规则启动期都会做处理器存在性、静态性、签名一致性与命中数自检——写错的规则在启动时炸，而不是在游戏里
静默地"少做一件事"。

### 极薄 Adapter

每个游戏版本只有一个薄适配层（`oml-adapter-*`），只负责钩子锚点与原生类型桥接。当前稳定主线是 **26.3**；
最新快照由 `oml-adapter-snapshot` 单独跟随迭代，26.4 正式版发布时适配即已完成大半。

### 双轨内容

- **代码轨**：`ContentRegistry` 声明方块 / 物品 / 配方，在注册表冻结点材料化为原生内容；
- **数据轨**：在 `.oml` 归档里的 `content.toml` 声明数据驱动的方块与物品，无需任何代码。

## 仓库结构

| 仓库                                                                 | 内容                                                       |
|----------------------------------------------------------------------|------------------------------------------------------------|
| [OhMyLoader](https://github.com/OhMyLoader/OhMyLoader)               | 加载器本体：内核、API、版本 adapter、安装器、原生库        |
| [OhMyLoaderGradle](https://github.com/OhMyLoader/OhMyLoaderGradle)   | 官方 Gradle 插件：运行环境装配与 `runClient` / `runServer` |
| [OhMyLoaderTestMod](https://github.com/OhMyLoader/OhMyLoaderTestMod) | 端到端验证模组，以外部 mod 工程的身份走完整消费路径        |

三个仓库互相只按 Maven 坐标消费。详细的模块划分见 loader 仓库 README。
