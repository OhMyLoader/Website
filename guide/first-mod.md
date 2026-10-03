# Your First Mod

## Prerequisites

- Zulu JDK 27;
- For now, local development requires publishing both repositories to your local Maven repository:
  the loader provides the runtime coordinates, the Gradle plugin provides `org.ohmyloader.gradle`.
  They are independent projects, so publish each one:

```bash
git clone https://github.com/OhMyLoader/OhMyLoader.git
git clone https://github.com/OhMyLoader/OhMyLoaderGradle.git
(cd OhMyLoader && ./gradlew publishToMavenLocal)
(cd OhMyLoaderGradle && ./gradlew publishToMavenLocal)
```

When only the plugin changed, republishing the plugin is enough.

## Set up the project

`settings.gradle.kts` — the plugin has to be resolvable from `mavenLocal` *before* the project
exists, and project-level `repositories { }` does not cover plugin resolution:

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

`build.gradle.kts`:

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

The Kotlin version has to be one that can target Java 27 class files; `2.5.0-Beta1` is what the
loader itself is built with.

The `oml-gradle` plugin does everything else: it puts `oml-api` on `compileOnly`, resolves the runtime
layer (including the per-version adapter) by version, provides `runClient` / `runServer`, and
downloads and verifies the game jar, libraries, natives and assets.

## Write the code

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

After the game finishes initializing, the loader instantiates the `@Mod` class and calls
`onInitialize`. Event subscription is reflection-free lambda registration.

## Run

```bash
./gradlew runClient   # runs the client
./gradlew runServer   # runs the server
```

`runClient` chains `fetchClientJar` / `fetchLibraries` / `extractNatives` / `fetchAssets` (all
SHA-1-verified) and installs this project's jar into `mods/`. The client's game directory is
`run/client/`, the server's is `run/server/`; their `assets/` and `mods/` directories are shared.

The server writes `run/server/eula.txt` on first start — for local development and automated
acceptance only; real deployments must read and accept the
[Minecraft EULA](https://aka.ms/MinecraftEULA) themselves.

## Tracking snapshots

Set the version to the literal `snapshot` — the runtime resolves it to the manifest's latest
snapshot, and a new snapshot needs no configuration change:

```kotlin
oml {
    minecraftVersion.set("snapshot")
}
```

The runtime layer then uses `oml-adapter-snapshot`, the working copy in the loader repository that
iterates with the snapshots. Its compatibility with the snapshot is continuously verified by the
loader's shape tests.
