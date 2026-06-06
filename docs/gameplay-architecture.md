# Gameplay architecture

## Managers: broad orchestration

Managers coordinate a domain and frequently own other objects. They are ideal for session-wide policies, queues, service access, or work not naturally attached to one unit.

Examples:

- `Managers.state.game_mode`: current ruleset and win/loss behavior.
- `Managers.state.conflict`: pacing, hordes, specials, and encounter direction.
- `Managers.state.spawn`: player/AI spawning coordination.
- `Managers.state.side`: parties/sides and allegiance.
- `Managers.state.network`: session-level network coordination.
- `Managers.player`: player records and local/remote player access.
- `Managers.backend`: inventory/progression service interface.

Before modifying a manager, locate its constructor and destructor to determine its lifetime and dependencies:

```bash
rg 'Managers\.state\.conflict =|ConflictDirector:new' scripts
rg 'Managers\.state\.conflict:|Managers\.state\.conflict\.' scripts
```

## Entity systems: ordered behavior over sets of units

`EntitySystemBag` owns an ordered set of systems. `ExtensionSystemBase` provides the common machinery for extension lists, update-function lists, freezing, hot join, and cleanup. Concrete systems can override creation/removal or run their own loops.

Typical systems include:

| Domain | Representative systems |
|---|---|
| AI | AI, navigation, group, slots, aggro, target override |
| Character simulation | locomotion, status, health, death, hit reaction, buff, talents |
| Combat/items | weapon, ammo, inventory, projectile, area damage, pickup |
| World interaction | interaction, doors, spawner, volume, objective, puzzle |
| Presentation/reaction | animation, audio, dialogue, outlines, HUD, world markers |
| Network | game object system and synchronized extension behavior |

Systems are not independent. For example, weapon behavior asks inventory/status/buff extensions for state; damage updates health; death notifies game mode/statistics/dialogue; HUD reads many extensions.

## Unit extensions: behavior attached to a native unit

A native `Unit` is the common identity shared by engine and Lua. Lua extensions add domain-specific behavior to that unit and are retrieved with `ScriptUnit` helpers.

Illustrative access:

```lua
local health = ScriptUnit.extension(unit, "health_system")
local status = ScriptUnit.extension(unit, "status_system")

if health and status and not status:is_disabled() then
    local current = health:current_health()
end
```

The exact extension keys and methods vary. Search existing consumers rather than guessing.

Extension lifecycle usually includes some subset of:

```lua
Extension.init(...)
Extension:update(unit, input, dt, context, t)
Extension:hot_join_sync(peer_id)
Extension:destroy()
```

Systems may enable/disable specific update functions dynamically to avoid iterating dormant behavior.

## Data-driven gameplay

The game commonly separates three pieces:

1. **Registry data**: a named entry, such as a breed, buff, action, weapon, damage profile, interaction, or pickup.
2. **Generic executor**: system/extension/helper code that reads the entry.
3. **Runtime context**: unit, owner, side, difficulty, network authority, current game mode, and time.

This makes data mutation a powerful modding technique, but entries are often shared. Clone before making a localized variant.

```lua
-- Illustrative pattern; table helpers/load timing depend on the mod framework.
local original = BuffTemplates.example_buff
local variant = table.clone(original)
variant.duration = 8
BuffTemplates.my_mod_example_buff = variant
```

Then register/reference the new name wherever the relevant subsystem expects it. If the name crosses the network, see [Networking and services](networking-and-services.md).

## AI stack

AI spans several layers:

```text
breed and action settings
  -> conflict director chooses what/when to spawn
  -> spawn manager/unit spawner creates AI unit
  -> AI/group/slot/navigation systems coordinate it
  -> blackboard stores runtime decision state
  -> behavior tree/action code selects and executes actions
  -> locomotion/animation/weapon/damage systems enact results
```

Useful search anchors are `BreedActions`, breed names, `BT*` classes/functions, `blackboard`, `Managers.state.conflict`, and the AI systems. Altering only an action table may not affect spawn composition, navigation, target selection, or animation resources.

### Example: reason about a stronger enemy

To make a breed tougher without replacing its whole behavior:

1. Locate the breed registry entry and where it selects health by difficulty.
2. Check whether health values are shared with other breeds.
3. Patch only the intended field after settings are loaded.
4. Test as host; clients should receive authoritative health state.
5. Check UI, stagger thresholds, damage profiles, and network lookup implications.

## Combat stack

Combat is assembled from many registries and extensions:

```text
input -> player/AI action -> weapon/action template
 -> hit detection -> damage profile -> DamageUtils
 -> buffs/talents/difficulty/armor rules
 -> health/death systems -> network/events/effects/statistics
```

Key areas include equipment settings, damage profiles, buff/talent definitions, weapon/projectile systems, health/death/hit-reaction systems, and helpers. A weapon can have first-person and third-person resources, action chains, ammo behavior, projectiles, effects, sounds, and UI metadata; changing one field rarely changes all of them.

## Game modes, mechanisms, mutators, and levels

- **Game mode** defines match rules and objectives.
- **Mechanism** selects/coordinates a broader experience and transitions.
- **Mutators** modify rules through reusable settings/logic.
- **Level settings** choose a level resource and configure game-mode/content behavior.
- **Terror events** script encounter sequences used by the conflict director.

These converge in `StateIngame`, where the selected level, mechanism, game mode, difficulty, network role, and managers are assembled.

## Events and cross-layer communication

Code communicates through several mechanisms:

- Direct calls through `Managers` or `ScriptUnit.extension`.
- Generic event managers for local publish/subscribe.
- Network RPCs for peer-to-peer/server-client events.
- Flow events/callbacks for level-authored graphs.
- Global registries/templates selected by name.
- State-machine transitions.

When adding behavior, choose the narrowest appropriate mechanism. A direct extension call is clearer for a local unit operation; an event is useful for loosely coupled local reactions; an RPC is required only for synchronized peer communication.

## Safe extension checklist

- Determine server/client ownership before mutating state.
- Determine the target object's lifetime.
- Reuse an existing registry/executor pattern.
- Avoid per-frame allocations in hot update loops.
- Do not retain destroyed units; validate with `Unit.alive` where appropriate.
- Pair registrations with unregister/cleanup.
- Test title -> keep -> mission -> keep transitions and hot join.
