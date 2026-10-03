# Introduction

**OhMyLoader (OML)** is a mod loader for Minecraft Java Edition: a self-developed bytecode
injection engine, a unified high-level mod API, and data-driven declarative content registration —
on both the client and the dedicated server. The current stable line is 26.3, and the whole stack
runs on Java 27.

## Design

### Unified semantic API

Mods program against high-level game semantics (events, lifecycle, content registration) and never
touch version internals. Cross-version differences are absorbed below the semantic layer, by the
per-version adapter, and never leak upward.

### Self-developed bytecode engine

The core ships its own lightweight bytecode surgery tooling, replacing the heavy SpongePowered Mixin
and Access Transformer layers:

- **Injection DSL**: anchors like `atHead` / `atReturn` / `beforeCall` combined with payloads like
  `call` / `modifyArg` / `redirectCall` — semantics aligned with Mixin, but purely declarative and
  self-checked at startup;
- **Mixin class merging**: `@Shadow` / `@Overwrite` / `@Unique` / `@Accessor` / `@Invoker` are all
  supported;
- **Annotation front-end**: developers used to Mixin can keep writing `@Mixin` / `@Inject`; the
  loader compiles the annotations into injection rules and runs them through the same pipeline.

Every rule is verified at startup — handler existence, staticness, signature consistency and match
counts. A wrong rule fails the launch, instead of silently doing one thing less in game.

### Thin adapters

Each game version gets one thin adapter module (`oml-adapter-*`) that only maps hook anchors and
native types. The current stable line is **26.3**; the latest snapshot is tracked separately by
`oml-adapter-snapshot`, so most of the adaptation for the 26.4 official release is already done.

### Two content tracks

- **Code track**: declare blocks / items / recipes through `ContentRegistry`; they are materialized
  into native content at the registry freeze point;
- **Data track**: drop a TOML file into `mods/` to declare data-driven blocks and items — no code.

## Repositories

| Repository                                                             | Contents                                                                              |
|------------------------------------------------------------------------|---------------------------------------------------------------------------------------|
| [OhMyLoader](https://github.com/OhMyLoader/OhMyLoader)                 | The loader: core, API, version adapters, installer, native library                    |
| [OhMyLoaderGradle](https://github.com/OhMyLoader/OhMyLoaderGradle)     | The official Gradle plugin: runtime assembly and `runClient` / `runServer`            |
| [OhMyLoaderTestMod](https://github.com/OhMyLoader/OhMyLoaderTestMod)   | The end-to-end verification mod, consuming OML exactly like an external mod project   |
