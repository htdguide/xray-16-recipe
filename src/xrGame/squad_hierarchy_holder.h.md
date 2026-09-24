# src/xrGame/squad_hierarchy_holder.h

> Declares the squad node of the team/squad/group hierarchy creatures are addressed by.

**Needs** — [`seniority_hierarchy_space.h`](seniority_hierarchy_space.h.md) · [`squad_hierarchy_holder_inline.h`](squad_hierarchy_holder_inline.h.md)
**Used by** — [`Spectator.cpp`](Spectator.cpp.md) · [`group_hierarchy_holder.cpp`](group_hierarchy_holder.cpp.md) · [`group_hierarchy_holder.h`](group_hierarchy_holder.h.md) · [`relation_registry_actions.cpp`](relation_registry_actions.cpp.md) · [`squad_hierarchy_holder.cpp`](squad_hierarchy_holder.cpp.md) · [`squad_hierarchy_holder_inline.h`](squad_hierarchy_holder_inline.h.md) · [`team_hierarchy_holder.cpp`](team_hierarchy_holder.cpp.md)
**Tier floor** — T3: a declaration over a fixed slot table

## Purpose

Declares the surface implemented in [`squad_hierarchy_holder.cpp`](squad_hierarchy_holder.cpp.md)
and [`squad_hierarchy_holder_inline.h`](squad_hierarchy_holder_inline.h.md).

## Exported units

- `group(group_id)` — the group node for an identifier, created on first ask.
- `team` — the parent node.
- `groups` — the raw slot table, for callers that iterate.
- Leadership accessors and re-derivation — compiled out; see the implementation twin.

## Notes

The thirty-two-group cap is declared here. It bounds the authored data, not the simulation,
and it is what lets a group identifier be a direct index into a table that never moves.
