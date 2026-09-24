# src/xrEngine/CustomHUD.h

> The interface through which the engine's frame loop drives the game's heads-up display, plus the bitset of HUD feature toggles.

**Needs** — [`EngineAPI.h`](EngineAPI.h.md) · [`EventAPI.h`](EventAPI.h.md) · [`pure.h`](pure.h.md) · [`device.h`](device.h.md)
**Used by** — [`r__dsgraph_build.cpp`](../Layers/xrRender/r__dsgraph_build.cpp.md) · [`r__dsgraph_render.cpp`](../Layers/xrRender/r__dsgraph_render.cpp.md) · [`CustomHUD.cpp`](CustomHUD.cpp.md) · [`FDemoRecord.cpp`](FDemoRecord.cpp.md) · [`IGame_Level.cpp`](IGame_Level.cpp.md) · [`xr_object_list.cpp`](xr_object_list.cpp.md) · [`HUDManager.cpp`](../xrGame/HUDManager.cpp.md) · [`HUDManager.h`](../xrGame/HUDManager.h.md) · [`UIDialogHolder.cpp`](../xrGame/UIDialogHolder.cpp.md) · [`UIGameCustom.cpp`](../xrGame/UIGameCustom.cpp.md) · [`UIGameCustom.h`](../xrGame/UIGameCustom.h.md)
**Tier floor** — T2: an interface plus a flag word; the flags are read by renderer and game code.

## Purpose

The heads-up display belongs to the game, but the frame loop must call it at two precise points in the render pass and must tell it about connection and level changes. This header is that contract — one of the small set of engine-declared interfaces the game module fills in, and the reason the engine can render a HUD without knowing what a weapon is.

## State

```text
RECORD HudFlags : bitset(32)
  crosshair, crosshair_distance, weapon, info, draw,
  crosshair_rt, weapon_rt, crosshair_dynamic, binocular_vision,
  crosshair_rt2, draw_rt, weapon_rt2, draw_rt2, left_handed
```

A single module-wide flag word, settable from the console and from the game's options screen. The `_rt` and `_rt2` variants are the same three toggles (crosshair, weapon, draw) once per renderer generation: the older forward renderer, the deferred renderer and the newest deferred renderer each ask a different question about whether the HUD is composited before or after tone mapping, and the shipped configuration sets them independently. They are separate bits, not a redundancy to collapse, because a player switching renderer generation keeps both settings.

The default has every render-target variant on and the three legacy non-`_rt` toggles off — the shipped default is a modern renderer with a dynamic crosshair.

## `HudInterface`

**Contract** — What the game must provide. It is also an event receiver and a UI-reset receiver, so the same object is registered on the engine's event queue and is told when the window resolution changes and the UI must re-lay out.

The frame loop calls, in this order per frame: `on_frame` during the simulation update, then `render_first` early in the render pass and `render_last` late in it. The two render points exist because the HUD is not one layer — the first-person weapon model is geometry that must be drawn inside the scene pass with its own near plane, while the crosshair and indicators are screen-space overlay drawn after. Both render entry points take a render-context identifier, because the renderer may be recording several contexts (the main view, a secondary view such as a scope or a mirror) and the HUD must know which it is drawing into.

```text
INTERFACE HudInterface
  render_first(context)            # in-scene layer: the first-person weapon
  render_last(context)             # overlay layer: crosshair, indicators
  on_frame()                       # simulation-side update
  load()                           # build from configuration, once
  on_connected() / on_disconnected()   # a network session began or ended
  render_active_item_ui()          # draw the UI painted onto the held item, if any
  render_active_item_ui_query() -> bool  # ...and whether there is one this frame
  on_object_released(object)       # drop every reference to a destroyed entity
```

**Invariants** — `on_object_released` must leave no reference to the named entity anywhere in the HUD before it returns. This is the engine-wide rule that a destroyed entity is unreferenced by every subsystem before its memory is released, and the HUD is one of the subsystems that holds entity pointers across frames.

**Notes** — The query-then-draw pair for the held item's own UI exists so the renderer can decide whether to allocate a render target for it before the draw begins; merging them would force the allocation every frame.
