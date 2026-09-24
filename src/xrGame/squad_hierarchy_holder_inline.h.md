# src/xrGame/squad_hierarchy_holder_inline.h

> Construction of a squad node — an empty slot table of fixed length and a link to the parent team — and the two accessors.

**Needs** — [`squad_hierarchy_holder.h`](squad_hierarchy_holder.h.md) · [`seniority_hierarchy_space.h`](seniority_hierarchy_space.h.md)
**Used by** — [`squad_hierarchy_holder.cpp`](squad_hierarchy_holder.cpp.md) · [`squad_hierarchy_holder.h`](squad_hierarchy_holder.h.md)
**Tier floor** — T3: field access

## Purpose

Definitions split out only because the original language wants inline bodies after the
class. A rebuild should fold them in.

## Constructor

**Contract** — records the parent team node, which may not be absent, and allocates the
group slot table at its full fixed length with every entry empty. No group node is built.

**Invariants** — allocating the table up front and filling it lazily is what makes a group
identifier a stable direct index. The table is never resized and never reordered.

## `team`, `groups`

**Contract** — the parent node, hard-failing if it is somehow absent, and the slot table
itself.
