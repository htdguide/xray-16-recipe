# src/xrEngine/IGame_ObjectPool.h

> Declares the client-object factory and its prefetch list.

**Needs** — [`IGame_ObjectPool.cpp`](IGame_ObjectPool.cpp.md) · [`xr_object.h`](xr_object.h.md)
**Used by** — [`IGame_ObjectPool.cpp`](IGame_ObjectPool.cpp.md) · [`IGame_Persistent.cpp`](IGame_Persistent.cpp.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md)
**Tier floor** — T2: a declaration of a factory and one list.

## Purpose

Declares the surface implemented in [`IGame_ObjectPool.cpp`](IGame_ObjectPool.cpp.md). The
type is embedded by value in the persistent layer (see
[`IGame_Persistent.h`](IGame_Persistent.h.md)), so its lifetime is that layer's lifetime.

## Exported units

- `create(name)` — build one client object from a configuration section name.
- `destroy(object)` — release one.
- `prefetch()` — instantiate one of everything the current game type declares, to warm the
  model and texture caches.
- `clear()` — release the prefetched set.
