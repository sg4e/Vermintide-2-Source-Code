# Networking and external services

## Server authority

Vermintide 2 gameplay generally treats the host/server as authoritative. Clients send requests/input-related events and receive authoritative outcomes. Dedicated-server paths also exist. Any engine-behavior mod must decide:

- Does this code run on server, client, or both?
- Who owns the state?
- Is the result purely visual/local or gameplay-consequential?
- How does a joining client reconstruct it?

A local HUD color can remain client-only. Damage, spawning, objective progress, inventory, or enemy AI normally cannot.

## RPC path

The core path is:

```text
sender code
  -> NetworkTransmit:send_rpc_* / RPC function
  -> native network channel (or queued local RPC)
  -> NetworkEventDelegate event table
  -> registered object's rpc_name(channel_id, ...)
```

`NetworkEventDelegate` allows multiple objects to register RPC callbacks and dispatches received events. Objects must unregister before destruction. `NetworkTransmit` handles server/client routing and queues local RPCs so local and remote behavior use a similar dispatch path.

### Illustrative existing-style receiver

```lua
local RPCS = { "rpc_my_event" }

function MySystem:init(context, name)
    self.network_event_delegate = context.network_event_delegate
    self.network_event_delegate:register(self, unpack(RPCS))
end

function MySystem:destroy()
    self.network_event_delegate:unregister(self)
end

function MySystem:rpc_my_event(channel_id, value_id)
    -- Validate authority and lookup values before applying.
end
```

Adding a truly new engine RPC generally requires schema/registration support beyond merely defining this method. For framework-level mod messages, prefer the mod networking facility described below.

## Network lookups and deterministic ordering

`scripts/network_lookup/network_lookup.lua` converts many string identifiers into compact numeric values: breeds, buffs, damage types, projectiles, levels, interactions, effects, game modes, and more. RPC definitions use these lookups and fixed types.

This creates a hard rule:

> If a mod changes a network-visible registry or lookup, every peer must construct exactly the same lookup, in the same order, before exchanging those values.

A mismatched index may decode as the wrong identifier rather than cleanly fail. Avoid appending networked entries only on one peer or at nondeterministic times.

## Unit identity and synchronized state

Native `Unit` references cannot simply be transmitted. Networked units have game-object IDs managed through unit storage/game-object systems. RPC handlers commonly receive an ID, resolve it through unit storage, validate that the unit exists/alive, then access extensions.

Hot join adds another requirement: systems/extensions that own persistent synchronized state implement `hot_join_sync(peer_id)` or reconstruct state from game objects/managers.

## Built-in mod networking

`ModManager` provides port-like mod messaging:

- `network_bind(port, callback)`
- `network_unbind(port)`
- `network_is_occupied(port)`
- `network_send(destination_peer_id, port, payload)`

The manager relays payloads through `rpc_mod_user_data` and tracks network context. Use a unique port and validate all payloads. This is usually safer than altering engine RPC schemas.

Illustrative concept:

```lua
local PORT = 4242

function mod:init()
    Managers.mod:network_bind(PORT, function(source_peer_id, payload)
        -- Decode defensively; never trust remote payloads.
    end)
end

function mod:on_unload()
    Managers.mod:network_unbind(PORT)
end
```

Check the current callback signature and your framework wrapper before shipping; payload size/type constraints are imposed by the existing RPC definition.

## Multiplayer-safe modification strategies

From safest to riskiest:

1. **Local presentation only:** UI layout, local sounds, debug overlays.
2. **Host-authoritative change using existing synchronized state:** host alters a value already replicated by the game.
3. **All-peers deterministic table patch:** every peer runs the same mod/version before lookup construction.
4. **ModManager message protocol:** all participating peers use a versioned custom payload protocol.
5. **Changing native/game RPC schema or lookup order:** high risk and generally unsuitable without full control of every peer and load order.

## Backend and platform layers

`scripts/managers/backend/` and `scripts/managers/backend_playfab/` expose inventory, progression, quests, entitlements, and service operations. `PlayFab/` contains service API wrappers. Other managers integrate matchmaking, Steam, account, EAC, presence, telemetry, and platform-specific behavior.

These are separate from simulation authority. A host-authoritative gameplay change does not automatically change persistent backend inventory/progression.

### Safety rules

- Do not treat backend responses as always immediate; many operations are asynchronous.
- Do not forge progression, entitlement, inventory, or service results.
- Respect offline/online backend differences.
- Avoid logging credentials, tickets, personal data, or full service payloads.
- Treat service interfaces as patch-sensitive and outside sanctioned gameplay modifications.

## Example: synchronized custom rule

Suppose all players with the mod should see a host-selected training multiplier:

1. Host owns the canonical multiplier.
2. On network context creation or peer join, host sends `{version, multiplier}` over a unique mod port.
3. Clients validate version, sender identity, type, and range.
4. Gameplay mutation occurs only on host; clients use the value for display unless existing engine replication requires a deterministic local calculation.
5. Unbind and clear session data on unload/state exit.

Do not allow arbitrary clients to set the host's multiplier.

## Network debugging checklist

- Test host, remote client, and dedicated-server paths where applicable.
- Test late join and reconnect.
- Confirm all registrations unregister on state exit/reload.
- Log sender peer IDs and message versions, not sensitive payloads.
- Verify server-only mutations do not also execute on clients.
- Verify lookup construction is deterministic.
