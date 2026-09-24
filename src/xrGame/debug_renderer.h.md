# src/xrGame/debug_renderer.h

> Declares the game layer's wireframe scratchpad, implemented in [`debug_renderer.cpp`](debug_renderer.cpp.md) and [`debug_renderer_inline.h`](debug_renderer_inline.h.md).

**Needs** — [`Include/xrRender/DebugRender.h`](../Include/xrRender/DebugRender.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md) · [`PHDebug.cpp`](PHDebug.cpp.md) · [`ai_stalker_debug.cpp`](ai/stalker/ai_stalker_debug.cpp.md) · [`dbg_draw_frustum.cpp`](dbg_draw_frustum.cpp.md) · [`debug_renderer.cpp`](debug_renderer.cpp.md) · [`debug_renderer_inline.h`](debug_renderer_inline.h.md) · [`doors_actor.cpp`](doors_actor.cpp.md) · [`game_sv_artefacthunt.cpp`](game_sv_artefacthunt.cpp.md) · [`level_debug.cpp`](level_debug.cpp.md) · [`player_hud_tune.cpp`](player_hud_tune.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CDebugRenderer`, the game layer's thin front end over the renderer's debug line
channel. The whole type — declaration, definition and every call site — exists only in debug
builds; a shipping rebuild can omit it entirely.

Exported units:

- `draw_line` — one segment, transformed.
- `draw_aabb` — an axis-aligned box from a centre and three half-extents.
- `draw_obb` — an oriented box, either with the half-extents folded into the matrix or
  supplied separately.
- `draw_ellipse` — a wireframe ellipsoid: a fixed unit sphere mesh pushed through the
  matrix.
- `render` — flush everything queued this frame.
- `add_lines` — the private funnel every shape goes through; hands a vertex array and an
  index-pair array to the renderer.

**Notes** — the level owns one instance and the shapes are queued per frame and flushed
once, so a caller anywhere in the game layer can draw without knowing anything about the
graphics device.
