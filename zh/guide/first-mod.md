# 第一个 mod

## 前置

- Zulu JDK 27；
- 本地开发暂时需要先把 loader 仓库发布到本地 Maven 仓库：

```bash
git clone https://github.com/OhMyLoader/OhMyLoader.git
cd OhMyLoader
./gradlew publishToMavenLocal
```

## 建立工程

`build.gradle.kts`：

```kotlin
plugins {
    kotlin("jvm") version "2.5.0"
    id("org.ohmyloader.gradle") version "0.1.0-SNAPSHOT"
}

repositories {
    mavenLocal()
    mavenCentral()
}

oml {
    minecraftVersion.set("26.3")
}
```

`oml-gradle` 插件会自动完成其余一切：把 `oml-api` 放上 `compileOnly`、按版本解析运行层（含 per-version
adapter）、提供 `runClient` / `runServer`、下载并校验游戏 jar / 运行库 / natives / 资产。

## 写代码

```kotlin
import org.ohmyloader.api.Mod
import org.ohmyloader.api.ModContext
import org.ohmyloader.api.OMLModInitializer
import org.ohmyloader.api.event.Events

@Mod(id = "my_mod", name = "My Mod", version = "1.0.0")
class MyMod : OMLModInitializer {
    override fun onInitialize(context: ModContext) {
        println("hello, ${context.id}!")

        Events.CHAT_RECEIVED.register { event ->
            if (event.message == "hello") {
                event.canceled = true
            }
        }
    }
}
```

游戏初始化完成后，加载器实例化 `@Mod` 类并调用 `onInitialize`；事件订阅是零反射的 lambda 注册。

## 运行

```bash
./gradlew runClient   # 运行客户端
./gradlew runServer   # 运行服务端
```

`runClient` 自动链式执行 `fetchClientJar` / `fetchLibraries` / `extractNatives` / `fetchAssets`（全部带
SHA-1 校验），本工程构建出的 jar 会被装进 `mods/`。游戏目录在 `run/client/`，服务端在 `run/server/`，
两者的 `assets/` 与 `mods/` 共享。

服务端首次启动会写 `run/server/eula.txt`——仅用于本地开发与自动化验收；真实部署请自行阅读并同意
[Minecraft EULA](https://aka.ms/MinecraftEULA)。

## 追快照

把版本改成字面量 `snapshot` 即可，运行期自动解析为版本清单里的最新快照，新快照发布无需改任何配置：

```kotlin
oml {
    minecraftVersion.set("snapshot")
}
```

此时运行层使用 `oml-adapter-snapshot`——它是 loader 仓库里跟随快照迭代的工作副本，钩子与快照的兼容性由
loader 仓库的形状测试持续验证。
