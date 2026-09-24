# src/xrGame/ai/monsters/poltergeist/poltergeist_movement.h

> Declares the poltergeist's own path follower, which while hidden moves a graph position rather than a body.

**Needs** — [`poltergeist_movement.cpp`](poltergeist_movement.cpp.md) · [`../control_path_builder.h`](../control_path_builder.h.md)
**Used by** — [`poltergeist.cpp`](poltergeist.cpp.md) · [`poltergeist_movement.cpp`](poltergeist_movement.cpp.md)
**Tier floor** — T2: substitutes for the shared follower at the point where it would hand a position to the character controller

## Purpose

Declares the surface implemented in
[`poltergeist_movement.cpp`](poltergeist_movement.cpp.md). It replaces exactly one operation of
the shared path follower — advancing along the current route — and inherits everything else.

## Exported units

- `move_along_path(controller, out destination, elapsed)` — the override. While the creature is
  visible it defers entirely to the shared implementation; while hidden it runs its own.
- `rendered_position` — the graph position lifted by the creature's current drift height. This
  is what the renderer and the physics layer are told, and it is never the same as the graph
  position while hidden.
