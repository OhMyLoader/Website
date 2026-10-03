# 第一个 mod

## 前置

- Zulu JDK 27；
- 本地开发暂时需要把两个仓库都发布到本地 Maven 仓库：loader 提供运行时坐标，Gradle 插件提供
  `org.ohmyloader.gradle`。两者是独立工程，各自发布一次：

```bash
git clone https://github.com/OhMyLoader/OhMyLoader.git
git clone https://github.com/OhMyLoader/OhMyLoaderGradle.git
(cd OhMyLoader && ./gradlew publishToMavenLocal)
(cd OhMyLoaderGradle && ./gradlew publishToMavenLocal)
```

之后只改动了插件时，重新发布插件即可。

## 建立工程

`settings.gradle.kts` —— 插件必须在工程存在**之前**就能从 `mavenLocal` 解析，而工程级的
`repositories { }` 管不到插件解析：

```kotlin
pluginManagement {
    repositories {
        mavenLocal()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositories {
        mavenLocal()
        mavenCentral()
    }
}

rootProject.name = "my-mod"
```

`build.gradle.kts`：

```kotlin
plugins {
    kotlin("jvm") version "2.5.0-Beta1"
    id("org.ohmyloader.gradle") version "0.1.0-SNAPSHOT"
}

kotlin {
    jvmToolchain(27)
}

oml {
    minecraftVersion.set("26.3")
}
```

Kotlin 版本必须能产出 Java 27 字节码，`2.5.0-Beta1` 正是 loader 自用的版本。

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
                event.cancel()
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
