# Content Registration

Content comes in two tracks: the **code track** (`ContentRegistry`, this page) and the **data
track** (TOML content packs, at the end). Both share one pipeline: declarations are collected
during startup and materialized into native content at the **registry freeze point**, after which
they are indistinguishable from vanilla content.

::: warning The registry freezes
Minecraft's registries freeze early in startup; afterward nothing can be added or removed. OML
materializes your declarations at that freeze point — after it (including at runtime) no new
content can appear. Declare content inside [`declareContent`](#declarecontent).
:::

## declareContent

Make the `@Mod` class implement `OMLContentProvider`, and the loader calls it before the game's
main logic starts:

```kotlin
@Mod(id = "my_mod", name = "My Mod", version = "1.0.0")
class MyMod : OMLModInitializer, OMLContentProvider {
    override fun declareContent(registry: ContentRegistry) {
        // declare here
    }
}
```

The registry is already bound to the mod's namespace: ids in declarations carry no domain, and OML
binds them under the mod id.

## Blocks

```kotlin
val block = registry.declareBlock("ruby_ore") {
    destroyTime = 3.0f
    explosionResistance = 6.0f
    requiresCorrectToolForDrops = true
}
```

| Property                        | Default                 | Meaning                            |
|---------------------------------|-------------------------|------------------------------------|
| `destroyTime`                   | vanilla default         | hardness (time to break)           |
| `explosionResistance`           | follows `destroyTime`   | blast resistance                   |
| `requiresCorrectToolForDrops`   | `false`                 | drops only with the correct tool   |

Every block automatically registers its block item. The returned `OMLBlock` exposes `platform`
(the native block object) once content registration completes — the escape hatch for
version-specific behavior.

## Items

```kotlin
registry.declareItem("ruby_sword") {
    maxDamage = 592
    attackDamage = 7.0
    attackSpeed = -2.4   // vanilla base 4.0 → effective 1.6
    miningSpeed = 1.0f
    toolDamagePerBlock = 2
    minesAndDrops("minecraft:stone", 8.0f)   // on stone: mining speed 8, drops loot
    overrideSpeed("minecraft:dirt", 4.0f)    // on dirt: mining speed 4, vanilla drops
    deniesDrops("minecraft:bedrock")         // never drops
}
```

| Property                       | Default              | Meaning                                                           |
|--------------------------------|----------------------|-------------------------------------------------------------------|
| `maxDamage`                    | none (unbreakable)   | max durability; > 0 shows the durability bar                      |
| `attackDamage`                 | none                 | main-hand attack damage **modifier** (vanilla fists: 1.0)         |
| `attackSpeed`                  | none                 | main-hand attack speed **modifier** (vanilla base: 4.0)           |
| `miningSpeed`                  | –                    | default mining speed when no rule matches (requires tool rules)   |
| `toolDamagePerBlock`           | `1`                  | durability cost per broken block                                  |
| `canDestroyBlocksInCreative`   | `true`               | instant breaking in creative mode                                 |

Tool rule semantics: `minesAndDrops` = matched blocks mine at the given speed **and drop loot**;
`overrideSpeed` = speed only, vanilla drops; `deniesDrops` = vanilla speed, never drops. These map
onto the native `DataComponents.TOOL`.

## Recipes and drops

### Crafting

```kotlin
// shaped: 2x2 ruby from 4 ruby items (pattern rows reference the key map, ' ' = empty cell)
registry.declareShapedCrafting(
    result = "my_mod:ruby_block",
    pattern = listOf("RR", "RR"),
    key = mapOf('R' to "my_mod:ruby"),
)

// shapeless: any arrangement of the ingredients
registry.declareShapelessCrafting(
    result = "my_mod:ruby",
    ingredients = listOf("my_mod:ruby_ore", "minecraft:stick"),
    count = 2,
)
```

Pattern rules: 1-3 rows of 1-3 cells, all rows the same width, every non-blank character must have
a key entry — a malformed pattern throws at declaration time, before the pack ever loads. Bare
ingredient ids are qualified with the mod's namespace. The datapack file id derives from the result
(`my_mod:ruby_block` → `my_mod_ruby_block`); two recipes with the same result get a numeric suffix.

### Smelting

```kotlin
// smelting (furnace: SMELTING / BLASTING / SMOKING, default SMELTING)
registry.declareSmelting(
    input = "minecraft:raw_iron",
    result = "minecraft:iron_ingot",
    experience = 0.7,
    cookingTime = 200,
)

// drop override: breaking ruby_ore drops 1–3 ruby
registry.declareBlockDrop(block = "ruby_ore", drop = "my_mod:ruby", dropCountMin = 1, dropCountMax = 3)
```

Recipes and loot tables materialize into datapack JSON served through OML's injected resource pack —
the **vanilla datapack reload path**, server-authoritative and synced to clients.

## The data track: TOML content packs

Content without code: drop a `.toml` file into the `mods/` directory (the file name becomes the
namespace) and declare blocks and items:

```toml
[block.ruby_ore]
destroy_time = 3.0
explosion_resistance = 6.0
requires_correct_tool = true

[item.ruby]
max_damage = 64
attack_damage = 2.0
mines_and_drops = "minecraft:stone"

[crafting.ruby_block]
type = "shaped"
pattern = ["RR", "RR"]
key = { R = "ruby" }

[crafting.ruby_from_ore]
type = "shapeless"
count = 2
ingredients = ["ruby_ore", "minecraft:stick"]
```

The fields map one-to-one onto the code API (`destroy_time` → `destroyTime`, …). A
`[crafting.<result-id>]` section declares a crafting recipe whose result is that item: `type`
selects shaped / shapeless; shaped needs `pattern` (rows) plus a `key` inline table mapping
pattern characters to item ids; shapeless needs an `ingredients` list; `count` is optional
(default 1). Validation errors (ragged patterns, unknown types, missing fields) fail the pack
with the reason. TOML packs share
the same collect → freeze-materialize pipeline as code mods, and asset injection treats the pack's
namespace like a mod domain. A misspelled field is called out by name in the startup log — nothing
is silently ignored.
## Distribution form: `.oml` archives

A content pack ships as a **`.oml` archive** (a plain zip renamed): `content.toml` at the root,
plus the pack's own `assets/` and `data/`:

```
MyPack.oml (zip)
├── content.toml
└── assets/
    └── my_pack/
        ├── textures/block/ruby_ore.png
        └── lang/en_us.json
```

Drop it into `mods/`. The archive name is the namespace; the archive's textures and lang files are
served through OML's injected resource pack.

## Hot reload

Recipes declared through the content tracks ride vanilla's datapack reload:

- **Textures / lang / models**: `F3+T` re-scans the mod jars and `.oml` archives — edited files are
  picked up on the spot.
- **Crafting recipes**: edit the recipe in the pack, then run `/reload` (or `F3+T`) — the loader
  re-reads the packs from disk and the datapack JSON is served fresh.
- **Blocks / items themselves**: not reloadable. The registries freeze early in startup; adding or
  changing a block or item requires a restart. This is a vanilla constraint, not an OML limitation.
