# Mod Entry & Events

## Mod entry

`@Mod` marks the mod's entry class, which implements the [`OMLModInitializer`](https://github.com/OhMyLoader/OhMyLoader) interface. After the
game finishes initializing, the loader instantiates the class and calls `onInitialize` with the
mod's own metadata:

```kotlin
@Mod(id = "my_mod", name = "My Mod", version = "1.0.0")
class MyMod : OMLModInitializer {
    override fun onInitialize(context: ModContext) {
        // context: id / name / version
    }
}
```

The `id` is all-lowercase and doubles as the mod's resource namespace (the default domain for
assets and content ids).

## Event registration

Events are registered as reflection-free lambdas on the `Events` object. The handler signature is
the event type; cancellable events are canceled by calling `cancel()` (`canceled` is a read-only
projection of it — reading it lets a later handler observe an earlier one's decision):

```kotlin
Events.CLIENT_TICK.register { /* every logical tick */ }
```

| Event                | Fired                                          | Cancellable   | Fields                                                             |
|----------------------|------------------------------------------------|---------------|--------------------------------------------------------------------|
| `CLIENT_TICK`        | every client logical tick                      | –             | –                                                                  |
| `SERVER_TICK`        | every server logical tick (dedicated server)   | –             | –                                                                  |
| `FRAME_RATE_LIMIT`   | every frame's frame-rate calculation           | –             | `currentLimit: Int`, writable `limit: Int` (overrides the limit)   |
| `GUI_OPEN`           | before any GUI is shown                        | ✅             | `screen: OMLScreen?` (null = the current GUI is being closed)      |
| `CHAT_SENT`          | the player sends a chat message                | ✅             | `message: String`                                                  |
| `CHAT_RECEIVED`      | a chat message is received (before display)    | ✅             | `message: String`                                                  |
| `WORLD_LOAD`         | entering a world / disconnecting               | –             | `world: OMLWorld?` (null = disconnected)                           |

### Examples

Capping the frame rate:

```kotlin
Events.FRAME_RATE_LIMIT.register { event ->
    if (event.limit > 120) event.limit = 120
}
```

Filtering chat:

```kotlin
Events.CHAT_RECEIVED.register { event ->
    if (event.message.contains("bad_word")) {
        event.cancel()
    }
}
```

Watching world joins and disconnects:

```kotlin
Events.WORLD_LOAD.register { event ->
    if (event.world == null) {
        println("left the world")
    }
}
```

::: tip
The dedicated server has no client event sources: `CLIENT_TICK`, `GUI_OPEN` and `FRAME_RATE_LIMIT`
only fire on the client, while `SERVER_TICK` only fires on the dedicated server. A mod built for
both sides needs no side checks of its own.
:::
