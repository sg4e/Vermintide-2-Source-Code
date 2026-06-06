# Modding engine behavior

## Understand the two mod layers

This source contains the game's built-in `ModManager`, which scans mod packages, loads their resource packages, executes their `run` function, and calls lifecycle callbacks on the returned object. Many community mods also use a higher-level framework (commonly VMF) that supplies convenient APIs such as hooks, commands, settings, localization, and network wrappers. The framework implementation/package format is not included here.

Do not assume a convenience API is native merely because existing community examples use it. Verify against your installed framework version.

## Built-in lifecycle visible in this source

A loaded mod object can receive:

```lua
mod:init(reload_data)
mod:update(dt)
mod:on_game_state_changed(status, state_name, state_object)
mod:on_reload()       -- return data to pass into init after reload
mod:on_unload()
```

`ModManager` catches callback errors and disables callbacks for the failing mod until reload. Developer mode supports a reload request. The manager also exposes custom mod messaging and print support.

Because mods load during boot, defer patches that target later globals or state managers until the relevant state enters.

## Choose an extension strategy

### 1. Observe first

Hook/read a method and log or display inputs/results without changing them. This reveals lifetime, authority, and frequency.

### 2. Patch data

Best when generic behavior already exists and a setting controls the desired result. Patch after the registry loads and before caches/lookups are built. Clone shared templates for localized behavior.

### 3. Hook a method

Best for inserting policy around existing behavior. Prefer a wrapper that calls the original exactly once. Avoid replacing an entire large function when a narrow helper is available.

### 4. Subscribe to an event/callback

Best for reactions that do not need to alter the producer. Pair registration and unregister.

### 5. Add a new system/extension

Powerful but invasive. It requires registration, creation context, update ordering, unit configuration, networking/hot join, and cleanup. Prefer hooking an existing system unless new per-unit state/update ownership is truly needed.

## Generic hook examples

These examples use a **conceptual VMF-style API** because the community framework is not present in this repository. Adapt names/signatures to your framework.

### Observe a manager method safely

```lua
mod:hook_safe(GameModeManager, "update", function(self, dt, t)
    -- Runs in addition to the original. Keep hot-path work tiny.
    if mod:get("debug_enabled") then
        mod.last_mode = self:game_mode()
    end
end)
```

Use a safe/post hook when you do not need to alter arguments or return values.

### Wrap and alter behavior

```lua
mod:hook(SomeClass, "some_method", function(func, self, value, ...)
    local adjusted = math.min(value * 1.10, 100)
    return func(self, adjusted, ...)
end)
```

Rules:

- Preserve `self`, varargs, and all return values.
- Call the original once unless intentionally replacing behavior.
- Check whether the method runs on server/client/both.
- Avoid recursive calls to the hooked method.
- Expect method signatures to change between patches.

### Replace behavior only when necessary

```lua
mod:hook_origin(SomeClass, "some_method", function(self, ...)
    -- Original is not called. You now own every invariant and side effect.
end)
```

Origin/replacement hooks are brittle because they bypass future fixes, events, network sends, statistics, and cleanup in the original.

## Example: state-aware diagnostic mod

```lua
local mod = get_mod("architecture_probe") -- framework-specific

function mod:on_game_state_changed(status, state_name, state_object)
    mod:echo("%s %s", status, state_name)

    if status == "enter" and state_name == "StateIngame" then
        local state = Managers.state
        mod:echo("server=%s mode=%s", tostring(state.network and state.network.is_server),
            tostring(state.game_mode and state.game_mode:game_mode()))
    end
end
```

This is a good first mod: it teaches state lifetime without mutating simulation.

## Example: alter an existing gameplay setting

Goal: adjust an existing buff only in a private/modded environment.

```lua
local mod = get_mod("buff_tuner")
local old_duration

local function patch()
    local template = BuffTemplates and BuffTemplates.some_existing_buff
    if template and old_duration == nil then
        old_duration = template.duration
        template.duration = 10
    end
end

function mod:on_game_state_changed(status, state_name)
    if status == "enter" and state_name == "StateIngame" then
        patch()
    end
end

function mod:on_unload()
    local template = BuffTemplates and BuffTemplates.some_existing_buff
    if template and old_duration ~= nil then
        template.duration = old_duration
    end
end
```

This illustrates restoration, but it may still be too late if the target is cached before `StateIngame`. Trace the target's require/consumer and choose the actual patch point.

## Example: alter host-authoritative behavior

```lua
mod:hook(SomeGameplaySystem, "apply_rule", function(func, self, unit, value, ...)
    if not self.is_server then
        return func(self, unit, value, ...)
    end

    if mod:is_enabled_for_current_mode() then
        value = value * 1.25
    end

    return func(self, unit, value, ...)
end)
```

The host guard prevents clients from independently mutating authoritative state. You must still verify that clients receive the result through existing replication and that the altered value is legal for the RPC/game-object schema.

## Example: custom mod message protocol

```lua
local PORT = 49123
local VERSION = 1

function mod:init()
    Managers.mod:network_bind(PORT, function(source_peer_id, payload)
        if type(payload) ~= "string" then return end
        local version, command = payload:match("^(%d+):([%w_]+)$")
        if tonumber(version) ~= VERSION then return end
        -- Validate sender/authority before acting on command.
    end)
end

function mod:on_unload()
    if Managers.mod:network_is_occupied(PORT) then
        Managers.mod:network_unbind(PORT)
    end
end
```

Use compact, versioned, validated payloads. Confirm exact callback and payload behavior in `ModManager` and your framework.

## Adding a new system: advanced outline

If a new per-unit system is unavoidable:

1. Define the extension class and its lifecycle/update methods.
2. Define a system derived from `ExtensionSystemBase` with the extension name(s).
3. Ensure both classes load before system construction.
4. Insert the system into `StateIngame`'s entity-system creation in a correct order.
5. Arrange extension creation data for target units.
6. Decide server/client behavior and register/unregister RPCs if necessary.
7. Implement hot-join sync for persistent network state.
8. Destroy extensions/system and release resources on state exit.

This approach is patch-sensitive because it changes central construction/order. A manager hook plus mod-owned table is often simpler.

## Tracing workflow for a real modification

Suppose you want to alter stagger behavior:

```bash
# Find definitions and broad consumers.
rg -n 'stagger' scripts/settings scripts/entity_system scripts/unit_extensions

# Find likely templates/lookup/network use.
rg -n 'stagger' scripts/network scripts/network_lookup

# Find constructors/update ordering for owning systems.
rg -n 'HitReactionSystem|Stagger' scripts/game_state scripts/entity_system

# Find DLC/level-specific overrides.
rg -n 'stagger' levels scripts/settings/dlcs
```

Then answer:

- Which setting chooses the value?
- Which helper/system consumes it?
- Is it server-authoritative?
- Is the value or template name serialized?
- Is behavior cached at spawn/init?
- What other systems react (animation, dialogue, effects, statistics)?

Only then choose data patch, hook, or replacement.

## Compatibility and safety

### Realm and fairness

Behavior-changing mods may be restricted to the modded realm/private play. The source includes mod metadata and compatibility/shim behavior; it should not be interpreted as permission to bypass anti-cheat, backend validation, sanctioned-realm rules, or multiplayer fairness controls.

### Patch resilience

- Hook narrow stable helpers, not giant state methods.
- Check symbols/fields before use and report a clear incompatibility.
- Avoid hard-coded update-order assumptions when an event exists.
- Namespace global additions and custom IDs.
- Keep mod protocol/version checks explicit.
- Re-test after every game patch.

### Cleanup

Track everything your mod owns:

```text
hooks (usually framework-managed)
event/RPC/mod-port registrations
resource package references
world/gui/unit objects
state-specific caches
mutated global values
```

Release or restore these during unload and state exit.

## Testing matrix

At minimum test:

| Dimension | Cases |
|---|---|
| Lifecycle | fresh launch, mod reload, title -> keep -> mission -> keep, shutdown |
| Role | host, remote client, bot-only/private session, dedicated server if relevant |
| Join | initial join, hot join, reconnect, host migration behavior if applicable |
| Content | base game and DLC absence/presence, multiple game modes/difficulties |
| Failure | missing target symbol, disabled mod, malformed network payload, target unit destroyed |
| Compatibility | with other hooks on the same method and after the latest patch |

## Anti-patterns

- Mutating `Managers.state.*` during boot without guards.
- Running authoritative damage/spawn logic independently on every client.
- Appending to a network lookup on only one peer.
- Replacing a large function merely to change one number.
- Holding native units/worlds/UI objects after state exit.
- Editing generated dialogue as if it were stable source.
- Assuming a referenced asset is loaded.
- Trusting custom network payloads.
- Using backend/service hooks to fabricate progression or entitlements.
