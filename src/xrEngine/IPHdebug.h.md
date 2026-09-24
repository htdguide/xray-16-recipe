# src/xrEngine/IPHdebug.h

> The port the physics module draws its debug triangles through, so it need not know the renderer.

**Needs** — [`Render.h`](Render.h.md)
**Used by** — [`phdebug.cpp`](phdebug.cpp.md) · [`PHDebug.cpp`](../xrGame/PHDebug.cpp.md)
**Tier floor** — T3: three calls forwarding geometry to a renderer

## Purpose

Physics debugging wants to draw contact manifolds, collision hulls and picked triangles.
Those shapes are produced deep inside the physics solve, which runs before and sometimes
between render passes, and they must persist on screen for longer than one frame to be
readable. This interface gives the physics module a way to emit them without a renderer
dependency, and gives the renderer a place to batch them.

Debug-only. A shipping rebuild may leave the whole port unfilled.

## State

One process-wide handle to the current implementation, filled when a renderer is created
and cleared when it is destroyed. Null until then, so every emit site must tolerate an
absent implementation.

```text
ph_debug_render : optional<PhDebugRender>   # global; none before the renderer exists
```

## `IPhDebugRender`

**Contract** — `open_cached_draw` and `close_cached_draw` bracket a batch of triangles;
everything emitted between them is retained and re-drawn for the given lifetime in
milliseconds, so a shape produced during one physics step stays visible across the many
frames a human needs to read it. `draw_tri` is only legal inside a bracket. Colours are
packed 32-bit with alpha; `solid` picks filled versus wireframe.

```text
INTERFACE PhDebugRender
  open_cached_draw()
  close_cached_draw(remove_after_ms : int)
  draw_tri(v0, v1, v2 : vector3, color : int (32-bit, packed), solid : bool)
```

**Invariants** — every `open` is matched by a `close`; `draw_tri` outside a bracket is a
programming error.
