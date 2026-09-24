# src/xrGame/seniority_hierarchy_holder_inline.h

> Construction and read-only access for the team registry.

**Needs** — [`seniority_hierarchy_holder.h`](seniority_hierarchy_holder.h.md) · [`seniority_hierarchy_space.h`](seniority_hierarchy_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3

## Purpose

Two bodies belonging to [`seniority_hierarchy_holder.h`](seniority_hierarchy_holder.h.md).

## Exported units

**Construction** — sizes the registry to its full capacity and fills every slot with
"absent". The registry is therefore *full-length and empty*, not empty-and-growing: the
team index is a direct slot index, so the container must already be that long before the
first lookup. This is the invariant the on-demand creation in
[`seniority_hierarchy_holder.cpp`](seniority_hierarchy_holder.cpp.md) rests on.

**`teams()`** — hands back the registry for read-only iteration, holes included. Callers
must skip absent slots.
