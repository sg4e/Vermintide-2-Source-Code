# Presentation, input, and developer tooling

## UI architecture

`scripts/ui/` contains several presentation layers:

- **Views:** menus and larger screens with their own lifecycle.
- **HUD UI:** in-game status, unit frames, crosshair, objectives, notifications, and overlays.
- **Widgets/scenes/helpers:** reusable definitions, animations, layout, and rendering support.
- **Atlas settings:** texture-atlas metadata and content-specific presentation.
- **Specialized presentation:** rewards, votes, tutorials, cutscenes, social wheel, DLC screens, and popups.

UI code typically reads managers/extensions but should not be the authoritative owner of gameplay state.

### UI interaction flow

```text
input manager/controller
 -> active view or in-game UI
 -> local UI state/animation
 -> manager/gameplay request when an action matters
 -> backend/network/game state responds
 -> UI refreshes from authoritative state
```

A mod that only changes rendering should hook as close to the UI component as possible instead of altering gameplay data.

## Input

Input managers abstract keyboard/mouse/controller and context. Active states/views can change input handling. Before binding a key, account for chat, menus, controller layouts, and existing actions. Avoid polling raw devices when the input manager/framework can provide a conflict-aware binding.

The built-in mod manager itself demonstrates raw keyboard polling for developer reload, but that is a narrow engine-level use and not necessarily the best mod UX.

## Camera, world, and rendering

- Native `World`, `Viewport`, `Camera`, `Gui`, and renderer APIs do the low-level work.
- Foundation script wrappers and managers coordinate lifetime.
- State/gameplay managers select camera behavior.
- Unit camera/first-person extensions attach behavior to player units.
- `core/stingray_renderer/` contains a small renderer visualization helper, not the renderer implementation.

Rendering changes that require new materials/shaders/assets cross the repository boundary and need proper resource packaging.

## Audio and dialogue

Audio spans:

- Native Wwise integration and Wwise worlds.
- `core/wwise/lua/` reference/Flow helpers.
- Game music/audio/sound systems and managers.
- Dialogue selection/context/surrounding-awareness.
- Sound settings and external sound banks.

A sound-event string is only a reference; the relevant bank/package must be loaded. Dedicated servers and disabled-render/audio contexts need guards.

## Effects and feedback

Hit effects, particles, decals, blood, outlines, world markers, camera effects, and HUD feedback translate simulation events into presentation. They often run client-side after a replicated authoritative event. This is a productive modding seam because visual changes can avoid changing simulation/network behavior.

## ImGui and debug tooling

`scripts/imgui/` provides developer/debug windows for inspecting systems and data. Foundation/game debug managers, performance managers, free flight, Testify hooks, and helper utilities offer additional introspection.

For a new diagnostic mod, prefer a read-only overlay:

```lua
-- Pseudocode: use the installed framework's draw API.
function mod:update(dt)
    local state = Managers.state
    local mode = state and state.game_mode
    if mode then
        mod:draw_text("Mode: " .. tostring(mode:game_mode()))
    end
end
```

Read-only diagnostics are easier to make multiplayer-safe and are excellent for discovering update ordering and ownership.

## Presentation mod examples

### Change a HUD component

1. Find visible text/material/widget name in `scripts/ui/hud_ui/` or relevant view.
2. Find the component's update/draw method.
3. Hook after update to alter only presentation state, or before draw to suppress/replace drawing.
4. Guard for alternate game modes, spectator/dead state, and resolution/UI scale.
5. Restore on unload or rely on framework-managed hooks.

### Add local feedback to an existing event

1. Hook an existing local event receiver or system method after it validates the event.
2. Do not mutate authoritative arguments.
3. Play an already-loaded sound/effect or display a UI message.
4. Rate-limit repeated events.
5. Guard dedicated server/no-render/no-audio contexts.

## Performance guidance

UI and presentation still run every frame. Avoid:

- Rebuilding large widget tables every update.
- Repeated global searches or `ScriptUnit.extension` calls when safe caching is possible.
- Formatting/logging every frame.
- Spawning GUI/world resources without cleanup.
- Holding references across state transitions.
