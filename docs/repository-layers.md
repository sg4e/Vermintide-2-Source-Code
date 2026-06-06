# Repository layers

## Top-level map

| Path | Layer | Responsibilities | Typical dependencies |
|---|---|---|---|
| `core/` | Native-engine Lua adapters | Small integrations for Wwise, navigation, renderer visualization, and volumetrics | Native engine APIs; loaded by engine resources/Flow rather than the whole gameplay stack |
| `foundation/` | Reusable Lua foundation | Class helper, callbacks, math/table utilities, state machines, package/world/time/event managers | Native primitives, but little Vermintide-specific gameplay |
| `scripts/` | Main game runtime | Boot, game states, managers, networking, gameplay systems/extensions, settings, UI, helpers | Every other layer |
| `levels/` | Level-authored Lua data | Level settings, terror events, patrols, conversations, pickups, ambient data, DLC level content | Settings registries, game modes, conflict director, dialogue, Flow |
| `dialogues/` | Generated dialogue content | Per-character/per-level dialogue rules and generated lookup content | Dialogue systems and resource packages |
| `PlayFab/` | Third-party/service client | PlayFab client/server/admin/matchmaker API wrappers and JSON/HTTP support | Backend managers and external PlayFab service |
| `backend/` | Backend constants | Backend error-code mapping | Backend managers/UI error handling |

The repository is overwhelmingly Lua. Asset names such as units, materials, sounds, packages, and levels are references to resources outside this checkout.

## `foundation/`: the lowest reusable Lua layer

### Utilities

`foundation/scripts/util/` supplies globally used primitives:

- `class.lua`: inheritance and class construction used by nearly every manager, state, and system.
- `callback.lua`: bound callbacks passed to asynchronous and event APIs.
- `state_machine.lua` and `visual_state_machine.lua`: generic state transition infrastructure.
- `script_world.lua`, `script_unit.lua`, `script_camera.lua`, and `script_viewport.lua`: safer/convenient wrappers around native objects.
- `table.lua`, `math.lua`, `vector3.lua`, `quaternion.lua`, `string.lua`, and queue/stack types: shared utility extensions.
- `application_parameter.lua`, `development_parameter.lua`, and `user_setting.lua`: configuration surfaces.
- `reportify.lua`, `crashify.lua`, `garbage_leak_detector.lua`, and `testify.lua`: diagnostics and automation.

### Foundation managers

`foundation/scripts/managers/` defines broad, reusable services:

- `WorldManager`: creates and tracks worlds.
- `TimeManager`: owns named timers and time scaling.
- `StateMachineManager`: tracks secondary state machines.
- `PackageManager`: reference-counts and asynchronously loads resource packages.
- `EventManager`: in-process publish/subscribe.
- `LocalizationManager`, `TokenManager`, `ReplayManager`, `FreeFlightManager`, and base player/chat types.

These are installed into the global `Managers` table by boot or a game state.

## `scripts/`: the game layer

### Boot and lifecycle

- `scripts/boot_init.lua` establishes platform/build globals, worlds, and global resource packages.
- `scripts/boot.lua` is the native host entry point. It loads foundational code, packages, mods, managers, settings, and the top-level `GameStateMachine`.
- `scripts/game_state/` contains title, loading, dedicated-server, and in-game states plus substates.

### Managers

`scripts/managers/` groups long-lived coordinators by domain. Important categories include:

- **Session/state:** network, player, side, spawn, game mode, transition, voting, difficulty.
- **Simulation/directing:** conflict director, talents, status effects, crafting, deeds, quests, weaves.
- **Platform/services:** backend, PlayFab backend, matchmaking, Steam, EAC, telemetry, account, presence.
- **Presentation:** input, camera, music, UI, popup, blood, decals, light effects.
- **Developer/mod:** debug, performance, Testify-adjacent tooling, mod manager.

Not every manager has the same lifetime. Check its constructor site before using it.

### Entity systems and unit extensions

- `scripts/entity_system/entity_system.lua` requires the concrete systems so their classes are available.
- `scripts/entity_system/entity_system_bag.lua` orders and updates active systems.
- `scripts/entity_system/systems/` contains systems such as health, buff, locomotion, AI, dialogue, inventory, weapon, projectile, interaction, and HUD.
- `scripts/unit_extensions/` contains behaviors attached to individual native `Unit` objects. Systems create, own, update, freeze, and remove these extensions.

### Data/settings

`scripts/settings/` is effectively a large data-definition layer. It defines or aggregates global registries for breeds, actions, weapons, items, buffs, talents, careers, levels, pickups, game modes, mutators, DLC, effects, and network-visible values. See [Data and content](data-and-content.md).

### Networking

- `scripts/network/` owns sessions, transmit/receive, RPC routing, clients/servers, unit storage/spawning, and profile synchronization.
- `scripts/network_lookup/` converts string-like gameplay identifiers to deterministic integer indexes used on the wire.

### Presentation and tooling

- `scripts/ui/` contains views, HUD components, widgets, scenes, and presentation helpers.
- `scripts/imgui/` contains developer/debug interfaces.
- `scripts/flow/` exposes Lua callbacks to authored level Flow graphs.
- `scripts/helpers/` and `scripts/utils/` contain game-specific shared logic.
- `scripts/tests/` contains the limited tests shipped with this source snapshot.

## `levels/` and `dialogues/`: authored/generated content layers

Level trees are organized by campaign/content family (`honduras`, DLC trees, inn, debug). A level may contribute settings, terror events, patrols, dialogue, and scripted data consumed by generic systems. `dialogues/generated/` is generated output; edit its source pipeline, when available, rather than assuming generated files are stable authoring surfaces.

## Dependency rules in practice

The dependency direction is mostly:

```text
native APIs <- foundation <- game runtime <- content definitions
```

But this is not a strict module system. Global registries and load order create implicit dependencies. A settings file can append to a table created by an earlier file; a system can call a global helper; level content can reference globally registered templates. Consequently:

- Load order matters.
- Naming collisions matter.
- Mutating a global table affects every consumer holding that table.
- Adding network-visible data can change deterministic lookup indexes.
- A mod should wait until its target global/class exists before patching it.

## Finding a layer from an unknown symbol

| Symbol shape | Likely owner |
|---|---|
| `Application.*`, `World.*`, `Unit.*`, `Gui.*`, `Wwise.*` | Native engine boundary |
| `Managers.foo` | Global manager/service |
| `Managers.state.foo` | active game-state manager/service |
| `FooSystem` | entity system, usually in `scripts/entity_system/systems/` |
| `FooExtension` | per-unit behavior, usually in `scripts/unit_extensions/` |
| `FooSettings`, `FooTemplates`, `FooLookup` | data registry/settings/network serialization |
| `StateFoo` | top-level or nested state machine state |
| `rpc_foo` | network event; search registrations and send sites |
| `FlowCallbacks.foo` / flow callback file | level Flow-to-Lua boundary |
