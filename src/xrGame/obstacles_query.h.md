# src/xrGame/obstacles_query.h

> Declares the accumulated obstacle set — a set of objects plus the union of the navigation vertices they block — implemented in [`obstacles_query.cpp`](obstacles_query.cpp.md).

**Needs** — [`obstacles_query_inline.h`](obstacles_query_inline.h.md) · [`Common/object_broker.h`](../Common/object_broker.h.md) · [`xrCore/Containers/AssociativeVector.hpp`](../xrCore/Containers/AssociativeVector.hpp.md)
**Used by** — [`moving_object.h`](moving_object.h.md) · [`moving_objects.h`](moving_objects.h.md) · [`moving_objects_static.cpp`](moving_objects_static.cpp.md) · [`obstacles_query.cpp`](obstacles_query.cpp.md) · [`obstacles_query_inline.h`](obstacles_query_inline.h.md) · [`refreshable_obstacles_query.h`](refreshable_obstacles_query.h.md) · [`restricted_object_obstacle.cpp`](restricted_object_obstacle.cpp.md) · [`restricted_object_obstacle.h`](restricted_object_obstacle.h.md) · [`static_obstacles_avoider.cpp`](static_obstacles_avoider.cpp.md) · [`static_obstacles_avoider.h`](static_obstacles_avoider.h.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `obstacles_query`, the thing a creature hands its pathfinder to say "these vertices
are blocked". It is a set of obstructing objects together with a lazily-computed, cached union
of the navigation vertices they cover, and a checksum that lets a caller ask cheaply whether
anything changed. Substance is split between
[`obstacles_query.cpp`](obstacles_query.cpp.md) and
[`obstacles_query_inline.h`](obstacles_query_inline.h.md).

Exported units:

- `obstacles_query` — the set, the cached vertex area, the checksum and the freshness flag.
  Non-copyable by assignment; copying is explicit.
- `add` — offer an obstructing object.
- `merge` (three forms) — union in another query's objects, with or without a proximity
  update.
- `set_intersection` — keep only the objects also present in another query.
- `remove_objects` — drop objects that no longer reach a given sphere.
- `remove_links` — drop one object, unconditionally.
- `objects_changed` / `refresh_objects` / `update_objects` — the staleness protocol.
- `area` — the union of blocked vertices, recomputed on demand.
- `obstacles` / `crc` / `actual` — the object set, the checksum, the freshness flag.
- `clear` / `swap` / `copy` — lifecycle.
- equality — compares by checksum and then by area.
