# src/xrServerEntities/xrServer_Space.h

> The enumeration of physics-object kinds, and the switch that adds the editor surface to every record in a tools build.

**Needs** — _(none)_
**Used by** — [`xr_object.h`](../xrEngine/xr_object.h.md) · [`GameObject.h`](../xrGame/GameObject.h.md) · [`actor_defs.h`](../xrGame/actor_defs.h.md) · [`alife_interaction_manager.h`](../xrGame/alife_interaction_manager.h.md) · [`alife_online_offline_group_brain.h`](../xrGame/alife_online_offline_group_brain.h.md) · [`alife_surge_manager.h`](../xrGame/alife_surge_manager.h.md) · [`memory_space.h`](../xrGame/memory_space.h.md) · [`sight_manager.h`](../xrGame/sight_manager.h.md) · [`smart_cover_loophole_planner_actions.h`](../xrGame/smart_cover_loophole_planner_actions.h.md) · [`alife_monster_brain.h`](alife_monster_brain.h.md) · [`xrServer_Objects_Abstract.h`](xrServer_Objects_Abstract.h.md)
**Tier floor** — T2: an enumeration and a build-configuration decision.

## Purpose

Almost nothing, but two of those nothings are load-bearing.

## State

```text
ENUM PhysicsObjectType    # serialized as a 32-bit field by the physic object record
  box            = 0
  fixed_chain    = 1
  free_chain     = 2
  skeleton       = 3      # the default; everything shipped is this
```

**Invariants** — the values are positional and reach disk, so they are frozen. Only
`skeleton` is used by shipped data; the other three describe a physics representation the
engine no longer builds. A rebuild reads the field and may refuse anything but `skeleton`.

## The editor-method switch

**Contract** — in a shipping build the record types declare no property-grid method; in every
other build they declare one. This is the *whole* mechanism by which this directory compiles
into both the game and the tools. A rebuild should express it as two compilations of one
data module with different capability sets — the records are identical either way, and the
editor surface is additive.

## Entity destruction helper

**Contract** — a one-line release of a record. Incidental: it exists because the tools and
the game free memory through different allocators and needed one name to call. A rebuild
with automatic lifetime deletes it.
