# src/xrGame/xrGame.h

> Declares the four-operation boundary between the engine and the game layer, and the two raw entity-factory entry points that cross it.

**Needs** — [`xrGame.cpp`](xrGame.cpp.md) · [`xrCore/clsid.h`](../xrCore/clsid.h.md) · [`xrEngine/EngineAPI.h`](../xrEngine/EngineAPI.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`xrGame.cpp`](xrGame.cpp.md) · [`entry_point.cpp`](../xr_3da/entry_point.cpp.md)
**Tier floor** — T1: declares symbols exported across a dynamic-library boundary with a fixed calling convention

## Purpose

Declares the surface implemented in [`xrGame.cpp`](xrGame.cpp.md). This is the *only* header
the engine includes from the game layer, which is the point: the boundary is four virtual
operations plus two free functions, and everything else in the chapter is private to it.

Exported units:

- `create_object(class identifier) -> object` — the entity factory's create entry.
- `destroy_object(object)` — the matching destroy entry.
- `initialize(create, destroy)` — bring the game layer up and publish those two.
- `finalize` — tear it down.
- `create_persistent` / `destroy_persistent` — the state that survives a level change.
- the module instance itself, exported by name so the engine can find it.

## State

`Stateless.` — one module instance, declared here and defined in
[`xrGame.cpp`](xrGame.cpp.md).

**Notes** — The two factory entries are declared with C linkage and an explicit calling
convention, and the module instance is exported by name. All three are the shape of "this
module may be a separately loaded library". The build can also link the game layer
statically, in which case the export machinery collapses to nothing and the same symbols are
resolved directly.

A rebuild has to decide whether the game layer is a loadable module. If it is, this is the
manifest of what must cross, and the constraints are the usual ones for a dynamic boundary:
a stable calling convention, no types whose layout the two sides might disagree on, and
allocation paired with deallocation on the same side. If it is not, the boundary is worth
keeping as an interface anyway — it is the seam a rebuild would use to replace the game
layer wholesale while keeping the engine.
