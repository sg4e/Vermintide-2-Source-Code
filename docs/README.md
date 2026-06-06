# Vermintide 2 Lua architecture guide

This directory is a map of the **Lua gameplay/runtime layer** published in this repository. It is intended for readers tracing behavior, debugging interactions, or designing a mod.

> **Scope warning:** this repository is not the complete native game engine or a buildable game checkout. The Lua code calls native Stingray/Fatshark APIs such as `Application`, `World`, `Unit`, `Network`, `Gui`, `Wwise`, and `ResourcePackage`; their implementations, most binary assets, build tools, and the launcher/mod-package format are not present. The `core/` tree here contains only a few Lua integrations. Treat undocumented native calls as an external engine boundary.

## Reading order

1. [Repository layers](repository-layers.md) — what each top-level tree and major `scripts/` subtree owns.
2. [Runtime lifecycle](runtime-lifecycle.md) — boot, game states, frame updates, loading, and shutdown.
3. [Gameplay architecture](gameplay-architecture.md) — managers, entity systems, unit extensions, AI, combat, and game modes.
4. [Data and content](data-and-content.md) — settings registries, levels, dialogue, DLC, Flow, and resource packages.
5. [Networking and services](networking-and-services.md) — authority, RPC dispatch, synchronized lookup tables, backend, platform, and telemetry.
6. [Presentation](presentation.md) — UI, HUD, input, camera, audio, rendering integration, and debug tooling.
7. [Modding engine behavior](modding.md) — extension points, hook strategy, examples, multiplayer safety, and a tracing workflow.

## The architecture in one picture

```text
Native host / Stingray callbacks
  init, update, render, shutdown
             |
             v
 scripts/boot.lua + scripts/boot_init.lua
  globals, packages, mods, global Managers, GameStateMachine
             |
             v
 Game states (title/loading/ingame/...)
             |
             +----------------------------+
             |                            |
             v                            v
 Global and state Managers          Content/data registries
 world, time, backend, input,       LevelSettings, breeds,
 network, game mode, conflict...    weapons, buffs, DLC...
             |
             v
 EntitySystemBag -> systems -> per-unit extensions
 AI / locomotion / health / weapon / dialogue / HUD / ...
             |
             +--> NetworkTransmit -> RPC -> NetworkEventDelegate
             +--> Flow callbacks / level scripts
             +--> UI, audio, effects, telemetry, backend
```

## Four ideas that make the code easier to navigate

1. **Most source files publish globals.** A `require("scripts/...")` commonly initializes a class or registry such as `StateIngame`, `BreedActions`, or `BuffTemplates` rather than returning a module.
2. **Managers own broad services; systems own unit-oriented update loops.** `Managers.state.game_mode` coordinates a match, while `HealthSystem` tracks health extensions attached to units.
3. **Behavior and data are deliberately separated.** Code frequently selects a named template from a global settings table, then executes generic behavior using that template.
4. **The server is authoritative for consequential gameplay.** Clients present and predict some behavior, but changes to damage, spawning, inventory, or synchronized templates must respect RPC schemas, lookup ordering, and server ownership.

## Fast tracing recipe

When investigating a feature, search in this order:

```bash
# 1. Find the setting/template name and all consumers.
rg 'my_template_name|MyTemplateTable' scripts levels

# 2. Find the manager/system that owns the behavior.
rg 'MyManager|MySystem|my_extension' scripts

# 3. Find registration, initialization, and update calls.
rg 'MySystem:new|add_system\(|MyManager:new|:update\(' scripts/game_state scripts/entity_system

# 4. Find network and content boundaries.
rg 'rpc_.*my_feature|NetworkLookup.*my_feature|my_feature' scripts/network scripts/network_lookup scripts/flow levels
```

## Documentation conventions

- Paths are relative to the repository root.
- Code snippets are illustrative and should be adapted to the installed mod framework and current game patch.
- “Native” means an API provided outside this Lua repository.
- “Global manager” means `Managers.<name>`; “state manager” means `Managers.state.<name>` and normally exists only during a game state such as `StateIngame`.
