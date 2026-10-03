# 内容注册

内容注册分两条轨：**代码轨**（`ContentRegistry`，本页）与**数据轨**（TOML 内容包，见文末）。
两者共用同一条管线：声明在启动期收集，在**注册表冻结点**一次性材料化为原生内容，之后与 vanilla
内容不可区分。

::: warning 注册表冻结
Minecraft 的注册表在启动早期冻结，冻结后不可增删。OML 在冻结点把声明的材料化进去——这之后
（包括运行期）都不能再新增内容。内容声明请写在 [`declareContent`](#declarecontent) 内。
:::

## declareContent

`@Mod` 类额外实现 `OMLContentProvider`，加载器会在游戏主逻辑启动前调用它：

```kotlin
@Mod(id = "my_mod", name = "My Mod", version = "1.0.0")
class MyMod : OMLModInitializer, OMLContentProvider {
    override fun declareContent(registry: ContentRegistry) {
        // 在这里声明
    }
}
```

`registry` 已绑定本 mod 的命名空间：声明里的 id 不带域名，OML 按 mod id 绑定。

## 方块

```kotlin
val block = registry.declareBlock("ruby_ore") {
    destroyTime = 3.0f
    explosionResistance = 6.0f
    requiresCorrectToolForDrops = true
}
```

| 属性                          | 默认               | 说明                   |
|-------------------------------|--------------------|------------------------|
| `destroyTime`                 | vanilla 默认       | 硬度（破坏耗时）       |
| `explosionResistance`         | 跟随 `destroyTime` | 爆炸抗性               |
| `requiresCorrectToolForDrops` | `false`            | 是否需要正确工具才掉落 |

每个方块自动注册对应的方块物品。返回的 `OMLBlock` 在内容注册完成后可访问 `platform`（原生方块对象）——
需要版本特定行为时的逃生舱。

## 物品

```kotlin
registry.declareItem("ruby_sword") {
    maxDamage = 592
    attackDamage = 7.0
    attackSpeed = -2.4   // vanilla 基础 4.0 → 实际 1.6
    miningSpeed = 1.0f
    toolDamagePerBlock = 2
    minesAndDrops("minecraft:stone", 8.0f)   // 对石头：挖掘速度 8 且掉落
    overrideSpeed("minecraft:dirt", 4.0f)    // 对泥土：挖掘速度 4，掉落同 vanilla
    deniesDrops("minecraft:bedrock")         // 永不掉落
}
```

| 属性                         | 默认           | 说明                                        |
|------------------------------|----------------|---------------------------------------------|
| `maxDamage`                  | 无（不可损坏） | 最大耐久，>0 时显示耐久条                   |
| `attackDamage`               | 无             | 主手攻击伤害 **加成值**（vanilla 空手 1.0） |
| `attackSpeed`                | 无             | 主手攻击速度 **加成值**（vanilla 基础 4.0） |
| `miningSpeed`                | –              | 无匹配规则时的默认挖掘速度（需有工具规则）  |
| `toolDamagePerBlock`         | `1`            | 每破坏一个方块的耐久消耗                    |
| `canDestroyBlocksInCreative` | `true`         | 创造模式能否瞬间破坏                        |

工具规则的语义：`minesAndDrops` = 匹配方块按指定速度挖掘 **且掉落战利品**；`overrideSpeed` = 只改速度、
掉落同 vanilla；`deniesDrops` = vanilla 速度且永不掉落。这些映射到原生的 `DataComponents.TOOL`。

## 配方与掉落

```kotlin
// 熔炼（furnace 可选 SMELTING / BLASTING / SMOKING，默认 SMELTING）
registry.declareSmelting(
    input = "minecraft:raw_iron",
    result = "minecraft:iron_ingot",
    experience = 0.7,
    cookingTime = 200,
)

// 方块掉落改写：破坏 ruby_ore 掉 1~3 个 ruby
registry.declareBlockDrop(block = "ruby_ore", drop = "my_mod:ruby", dropCountMin = 1, dropCountMax = 3)
```

配方与掉落表材料化为数据包 JSON，经 OML 注入的资源包走 **vanilla 自己的数据包重载路径**——
服务端权威、自动同步到客户端。

## 数据轨：TOML 内容包

不需要写代码的内容：在 `mods/` 目录放一个 `.toml` 文件（文件名即命名空间），声明方块与物品：

```toml
[block.ruby_ore]
destroy_time = 3.0
explosion_resistance = 6.0
requires_correct_tool = true

[item.ruby]
max_damage = 64
attack_damage = 2.0
mines_and_drops = "minecraft:stone"
```

字段与代码 API 一一对应（`destroy_time` → `destroyTime` …）。TOML 包与代码 mod 共享同一条
收集 → 冻结材料化管线，资产注入也把 TOML 包的命名空间当作 mod 域对待。字段拼错会在启动日志里
被逐条点名——不存在静默忽略。
