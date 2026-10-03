# mod 入口与事件系统

## mod 入口

`@Mod` 标记模组入口类，实现 [`OMLModInitializer`](https://github.com/OhMyLoader/OhMyLoader) 接口。
游戏初始化完成后，加载器实例化该类并调用 `onInitialize`，传入模组自身的元数据：

```kotlin
@Mod(id = "my_mod", name = "My Mod", version = "1.0.0")
class MyMod : OMLModInitializer {
    override fun onInitialize(context: ModContext) {
        // context: id / name / version
    }
}
```

`id` 全小写，也是这个 mod 的资源命名空间（资产、内容 id 的默认域）。

## 事件注册

事件通过 `Events` 对象的函数式 lambda 注册，零反射、可追溯。处理器签名即事件类型，取消类事件直接置
`canceled = true`：

```kotlin
Events.CLIENT_TICK.register { /* 每逻辑帧 */ }
```

| 事件               | 触发时机                        | 可取消 | 字段                                                   |
|--------------------|---------------------------------|--------|--------------------------------------------------------|
| `CLIENT_TICK`      | 客户端每逻辑 tick               | –      | –                                                      |
| `SERVER_TICK`      | 服务端每逻辑 tick（专用服务端） | –      | –                                                      |
| `FRAME_RATE_LIMIT` | 每帧帧率上限计算                | –      | `currentLimit: Int`、可写 `limit: Int`（改写帧率上限） |
| `GUI_OPEN`         | 任意 GUI 显示前                 | ✅     | `screen: OMLScreen?`（null = 当前 GUI 正在关闭）       |
| `CHAT_SENT`        | 玩家发送聊天消息                | ✅     | `message: String`                                      |
| `CHAT_RECEIVED`    | 收到聊天消息（显示前）          | ✅     | `message: String`                                      |
| `WORLD_LOAD`       | 进入世界 / 断开连接             | –      | `world: OMLWorld?`（null = 断开）                      |

### 示例

修改帧率上限：

```kotlin
Events.FRAME_RATE_LIMIT.register { event ->
    if (event.limit > 120) event.limit = 120
}
```

拦截聊天：

```kotlin
Events.CHAT_RECEIVED.register { event ->
    if (event.message.contains("bad_word")) {
        event.canceled = true
    }
}
```

监听进世界与断开：

```kotlin
Events.WORLD_LOAD.register { event ->
    if (event.world == null) {
        println("left the world")
    }
}
```

::: tip
专用服务端没有客户端事件源：`CLIENT_TICK` / `GUI_OPEN` / `FRAME_RATE_LIMIT` 等只会在客户端触发，
`SERVER_TICK` 只在专用服务端触发。同一份 mod 源码双端运行时无需自行判边。
:::
