# src/xrGame/stalker_velocity_holder.h

> Declares the shared registry of per-section speed tables, and its process-wide handle.

**Needs** — [`stalker_velocity_holder.cpp`](stalker_velocity_holder.cpp.md) · [`stalker_velocity_collection.h`](stalker_velocity_collection.h.md) · [`stalker_velocity_holder_inline.h`](stalker_velocity_holder_inline.h.md)
**Used by** — [`stalker_movement_manager_base.cpp`](stalker_movement_manager_base.cpp.md) · [`stalker_velocity_holder.cpp`](stalker_velocity_holder.cpp.md) · [`stalker_velocity_holder_inline.h`](stalker_velocity_holder_inline.h.md)
**Tier floor** — T2: a keyed cache and the single handle to it.

## Purpose

Declares the surface implemented in
[`stalker_velocity_holder.cpp`](stalker_velocity_holder.cpp.md), plus the accessor that
makes the registry reachable from anywhere a stalker is constructed. It is one of the
engine's process-wide singletons; see
[`stalker_velocity_holder_inline.h`](stalker_velocity_holder_inline.h.md) for how it comes
into being, which is the only decision here.

## Exported units

- `collection(section)` — the shared table for a configuration section, loading on demand.
- `stalker_velocity_holder()` — the process-wide registry, created on first use.
