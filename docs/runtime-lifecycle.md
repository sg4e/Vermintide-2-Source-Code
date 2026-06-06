# Runtime lifecycle and layer interaction

## 1. Native host enters Lua

The external engine loads `scripts/boot.lua`, which executes `scripts/boot_init.lua`. The Lua file publishes the conventional host callbacks:

```lua
function init()       -- initialize Boot
function update(dt)  -- boot update, then game update
function render()     -- boot/loading or game rendering
function shutdown()   -- destroy state and managers
```

This is the topmost Lua/native seam. The native host supplies frame delta, rendering, worlds, units, input devices, networking, and resources.

## 2. `boot_init.lua` establishes process-wide context

Early initialization:

- Imports native `s3d` symbols into `_G` when present.
- Detects `BUILD`, `PLATFORM`, console/Windows/Linux flags, Steam, dedicated server, and launch mode.
- Creates the optional global music world and Wwise world.
- Creates `script_data` from application settings/build information.
- Defines the platform-specific `GlobalResources` package list and asynchronous load handling.

This phase runs before ordinary game services exist, so code here uses native APIs and globals directly.

## 3. Boot loads foundation and global services

`Boot:booting_update(dt)` progresses through startup states instead of blocking one frame. In broad terms it:

1. Loads boot resource packages.
2. Loads foundation utilities and creates `Managers.package`.
3. Initializes development/application parameters.
4. Loads global resources.
5. Creates Curl and `ModManager`, then waits for enabled mods to load.
6. Requires game settings/classes and initializes process-wide managers.
7. Creates the top-level `GameStateMachine`.
8. Switches rendering/updating from boot behavior to game behavior.

Mods load deliberately early, before the full game initialization completes. This lets a mod alter classes and registries, but it also means some globals do not exist during the mod object's `init` callback.

## 4. Global managers vs state managers

The shared `Managers` table is a service locator.

```lua
Managers.time          -- process/global lifetime
Managers.world         -- process/global lifetime
Managers.package       -- process/global lifetime
Managers.backend       -- generally front-end/global service

Managers.state.network -- created for the active in-game state
Managers.state.entity  -- created for the active in-game state
Managers.state.game_mode
Managers.state.conflict
```

A common failure is calling `Managers.state.*` from the title screen or during loading. Defensive code should verify both `Managers.state` and the desired manager.

```lua
local game_mode_manager = Managers.state and Managers.state.game_mode
if game_mode_manager then
    local mode = game_mode_manager:game_mode()
end
```

## 5. The top-level game state machine

`GameStateMachine` derives from the foundation state machine and wraps state changes with mod notifications. Its states include splash/title/loading/in-game/dedicated-server paths. A transition roughly follows:

```text
old state on_exit/destroy
  -> optional mod callback: on_game_state_changed("exit", ...)
  -> instantiate/enter new state
  -> optional mod callback: on_game_state_changed("enter", ...)
```

State objects own the services whose lifetime matches that state. `StateIngame:on_enter`, for example, creates the network event delegate and many `Managers.state` services, sets up the level/session, and creates entity systems. `StateIngame:on_exit` tears them down.

## 6. A game frame

At a high level, a running frame is layered like this:

```text
native update(dt)
  -> Boot:game_update(dt)
     -> timers/global managers
     -> current GameStateMachine state
        -> StateIngame:update(dt, main_t)
           -> receive/process network events
           -> state managers/directors
           -> entity-system pre_update/update/post_update
           -> UI/presentation and queued transitions
     -> ModManager:update(dt) / mod update callbacks

native render()
  -> state pre_render
  -> WorldManager:render()
  -> state render/post_render
  -> URL loader post-render work
```

Exact ordering matters. If a hook observes stale data, identify whether the producer runs in pre-update, update, post-update, or an RPC callback.

## 7. Entity-system lifecycle inside `StateIngame`

`StateIngame` creates an `EntitySystemBag` and adds concrete systems in a deliberate order. The bag maintains separate update lists, invokes systems, performs hot-join synchronization, and destroys systems in reverse order. Systems receive a creation/update context containing shared services such as world, server status, network delegates, unit storage, and statistics.

A unit generally enters gameplay like this:

```text
unit resource spawned in native World
  -> UnitSpawner / GameObjectSystem assigns network identity if needed
  -> EntityManager adds configured extensions
  -> owning ExtensionSystemBase creates extension instance
  -> ScriptUnit stores extension by system/name
  -> system invokes extension pre_update/update/post_update
  -> removal reverses registration and destroys extension
```

## 8. Loading and resource packages

Lua refers to assets by resource path, but `PackageManager`/`ResourcePackage` make them available. Package loading is asynchronous and reference-based. A behavior that spawns a unit, plays an effect, opens a UI scene, or loads dialogue must ensure the corresponding package is loaded for the current platform/state.

Do not confuse:

- `require("scripts/foo")`: load/execute Lua code.
- `Managers.package:load(...)` / `ResourcePackage`: load engine assets.
- `World.spawn_unit(...)`: instantiate a loaded unit resource.

## 9. Shutdown and state transition hygiene

Every registration must have an inverse:

- Network delegate `register` -> `unregister`.
- Event callback registration -> removal.
- Package load/reference -> unload/release.
- Manager/system creation -> destroy.
- Hook or global mutation -> restore when a reloadable mod unloads, if the framework does not do so.

State transitions and mod reloads expose leaks quickly. Avoid retaining units, worlds, GUI objects, or manager instances after their owning state exits.

## Example: tracing damage through the layers

A simplified trace is:

1. Weapon/action settings choose attack and damage-profile names.
2. A weapon/action extension detects a hit using native physics/unit APIs.
3. Damage helpers resolve the profile, armor, buffs, talents, difficulty, and friendly-fire rules.
4. Server-authoritative health/damage systems mutate the victim extension and may emit RPCs.
5. Death, statistics, dialogue, effects, HUD, and telemetry systems react.
6. UI/audio/render layers present the result.

To investigate a damage-profile change:

```bash
rg 'damage_profile_name' scripts/settings scripts/unit_extensions scripts/entity_system
rg 'rpc_.*damage|rpc_.*hit' scripts
rg 'DamageProfile|DamageUtils|HealthSystem' scripts
```
