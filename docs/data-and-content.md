# Data, content, and authored layers

## Settings are executable Lua registries

`scripts/settings/` contains Lua files that create and extend global data tables. They are not passive JSON configuration: they can require other files, calculate values, append DLC data, define callbacks, and mutate shared registries.

Major families include:

- Breeds, AI actions, and enemy composition.
- Equipment, items, weapons, actions, damage profiles, traits, properties, and pickups.
- Buffs, talents, careers, profiles, and status effects.
- Levels, game modes, mechanisms, difficulty, mutators, and terror events.
- UI atlases/colors/fonts and effect/sound mappings.
- DLC-specific registries and conditional loading.
- Network-visible constants and lookup source tables.

### Registry aggregation pattern

A common pattern is:

```text
base settings file creates GlobalRegistry
  -> requires base entries
  -> DLC settings require/append additional entries
  -> network lookup builds deterministic list from final registry
  -> systems select an entry by string/index at runtime
```

This makes **when** a mod mutates data important. Too early and the registry does not exist; too late and derived lookup tables or cached data already exist.

## DLC and feature loading

DLC settings control optional requires and content registration. Do not assume every content file loads in every installation or game mode. Search for the target file's require site and its DLC gate.

```bash
rg 'require\(".*target_file|target_file' scripts/settings scripts/boot.lua
rg 'DLCSettings|dlc_name' scripts/settings
```

A robust mod checks that optional tables/entries exist before patching them.

## Levels

`levels/` contains authored Lua associated with maps and content families. Typical level content includes:

- Level metadata/settings and package references.
- Terror events and scripted encounter sequences.
- Patrol and spawn data.
- Level-specific conversations/dialogue rules.
- Pickups, ambient behavior, and objective data.

Generic runtime systems consume this content. For example, the conflict director executes terror-event data; dialogue systems consume dialogue rules; level Flow invokes callbacks exposed by `scripts/flow/`.

### Example: trace a level encounter

```bash
# Find event definition and references.
rg 'event_name' levels scripts/settings/terror_events scripts

# Find the generic executor and related flow/RPC hooks.
rg 'TerrorEvent|terror_event|rpc_.*event' scripts/managers/conflict_director scripts/flow scripts/network
```

Changing a level script may require resources not included in this repository and may make the level incompatible with unmodded peers.

## Dialogue

`dialogues/`, particularly `dialogues/generated/`, contains generated dialogue rule data. The dialogue architecture combines:

1. Dialogue settings/rules and voice-event resource names.
2. Dialogue context attached to units.
3. Surrounding-aware events that announce relevant world events.
4. Dialogue system selection/playback.
5. Wwise/native audio playback and subtitle/UI presentation.

Generated files are useful references but fragile edit targets. Prefer registering or patching at a stable runtime boundary, and ensure required audio resources are packaged.

## Flow: authored graph to Lua boundary

Stingray Flow graphs can call functions exposed in `scripts/flow/` and the small `core/*/lua/*flow_callbacks.lua` adapters. Flow is commonly used for level scripting and engine-facing authored behavior.

Conceptually:

```text
level Flow node/event
  -> Lua flow callback
  -> manager/system/native API
  -> optional output/event back to Flow
```

Before changing a Flow callback, search for all callback names and expected output pins. Flow callers are assets outside this repository, so Lua search results may not reveal every caller.

## Resource packages and native assets

Most settings use string paths to resources. Lua presence does not imply the asset is loaded or included. Common resource types include:

- Unit/level resources.
- Textures, materials, shaders, and UI atlases.
- Wwise banks/events.
- Animation/state-machine resources.
- Particle/effect resources.
- Resource packages that group these assets.

The package layer is part of behavior: loading too late causes missing resources; never unloading leaks memory; loading platform-incompatible assets can fail.

## Data mod example: changing an existing template

Use a post-load hook/callback supplied by your mod framework, then patch the narrowest field:

```lua
-- Illustrative only: verify names and shape in the current source snapshot.
local template = BuffTemplates.some_existing_buff
if template then
    template.duration = 12
end
```

Risks:

- The table may be cached/copied before your mutation.
- The entry may be shared by player and enemy behavior.
- A server-only mutation may disagree with clients.
- Sanctioned/unmodded realms may prohibit the behavior.

## Data mod example: adding a variant safely

```lua
local base = BuffTemplates.some_existing_buff
if base and not BuffTemplates.my_mod_variant then
    local variant = table.clone(base)
    variant.duration = 12
    BuffTemplates.my_mod_variant = variant
end
```

Then reference `my_mod_variant` only from your new/altered behavior. If the identifier is encoded through `NetworkLookup.buff_templates`, every peer must construct the same lookup in the same order before networking starts.

## Content-layer checklist

- Is the file base, DLC-gated, level-specific, or generated?
- Is the registry complete when you patch it?
- Is a derived cache/lookup built afterward?
- Does the asset exist and is its package loaded?
- Does the string cross the network?
- Do all peers have identical content?
