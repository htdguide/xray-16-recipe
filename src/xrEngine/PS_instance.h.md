# src/xrEngine/PS_instance.h

> Declares the base every live particle effect in the world derives from.

**Needs** — [`PS_instance.cpp`](PS_instance.cpp.md) · [`ISheduled.h`](ISheduled.h.md) · [`IRenderable.h`](IRenderable.h.md) · [`xrCDB/ISpatial.h`](../xrCDB/ISpatial.h.md)
**Used by** — [`IGame_Persistent.cpp`](IGame_Persistent.cpp.md) · [`PS_instance.cpp`](PS_instance.cpp.md) · [`ParticlesObject.cpp`](../xrGame/ParticlesObject.cpp.md) · [`ParticlesObject.h`](../xrGame/ParticlesObject.h.md)
**Tier floor** — T2: three facets composed plus a lifetime counter

## Purpose

Declares the surface implemented in [`PS_instance.cpp`](PS_instance.cpp.md).

A particle instance wears all three object facets — spatial (so visibility finds it),
scheduled (so its life ticks down), renderable (so it is drawn) — but is *not* a game
object: it has no network identity, no save state and no script presence. This class is
what separates "a thing in the world" from "a thing the game simulates".

Exported units:

- `CPS_Instance` — the base: registers itself in the persistent layer's active set at
  construction and removes itself at destruction, counts its life down each scheduled
  update, and requests its own deferred deletion when the count runs out.
- `PSI_destroy`, `PSI_alive`, `PSI_IsAutomatic`, `PSI_SetLifeTime` — the lifetime surface.
- `Play`, `Locked` — what a concrete effect must supply: how to start, and whether it is
  currently pinned open by something else.
- `destroy_on_game_load` — whether this effect must be wiped when a level is loaded.
