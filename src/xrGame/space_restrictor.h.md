# src/xrGame/space_restrictor.h

> Declares the restrictor client object: an invisible volume entity, base class of every zone and smart terrain.

**Needs** — [`GameObject.h`](GameObject.h.md) · [`space_restrictor_inline.h`](space_restrictor_inline.h.md) · [`xrServerEntities/restriction_space.h`](../xrServerEntities/restriction_space.h.md)
**Used by** — [`CustomZone.cpp`](CustomZone.cpp.md) · [`CustomZone.h`](CustomZone.h.md) · [`script_game_object.cpp`](script_game_object.cpp.md) · [`script_game_object3.cpp`](script_game_object3.cpp.md) · [`script_zone.cpp`](script_zone.cpp.md) · [`script_zone.h`](script_zone.h.md) · [`smart_zone.h`](smart_zone.h.md) · [`space_restriction_holder.cpp`](space_restriction_holder.cpp.md) · [`space_restriction_shape.cpp`](space_restriction_shape.cpp.md) · [`space_restrictor.cpp`](space_restrictor.cpp.md) · [`space_restrictor_inline.h`](space_restrictor_inline.h.md) · [`space_restrictor_script.cpp`](space_restrictor_script.cpp.md)
**Tier floor** — T2: a declaration over a cached primitive list

## Purpose

Declares the surface implemented in [`space_restrictor.cpp`](space_restrictor.cpp.md) and
[`space_restrictor_inline.h`](space_restrictor_inline.h.md). It is one of the most widely
derived-from types in the chapter: anomalies, damaging zones, campfires and smart terrains
are all restrictors, so the lifecycle and the containment test declared here are inherited
across most of the world's non-creature entities.

## Exported units

- `net_Spawn`, `net_Destroy` — the entity lifecycle, including registration with the restriction registry.
- `inside(sphere)` — exact containment against the union of the authored primitives.
- `Center`, `Radius` — world-space bounds.
- `UsedAI_Locations` — constantly false; a restrictor is not an obstacle on the mesh.
- `register_schedule` — constantly false; a restrictor is never updated.
- `spatial_move` — invalidates the cached world-space geometry.
- `restrictor_type`, `actual` — accessors.
- `cast_restrictor` — narrows a game object to a restrictor without a language cast, for the many call sites that must ask "is this thing a volume".
- `OnRender` — debug visualization.
- Script registration — see [`space_restrictor_script.cpp`](space_restrictor_script.cpp.md).

## Notes

The cached world-space primitives are declared mutable so that containment can be asked of a
const object, which is what the restriction family needs. That is an artifact of the
original language; the decision underneath is that the cache is derived state and does not
count as the object changing.

A box is stored as exactly six planes, which is the representation the containment test
wants; the count is fixed because the primitives are boxes, not general convex hulls.
