# src/xrGame/team_hierarchy_holder.h

> Declares the middle level of the team/squad/group hierarchy.

**Needs** — [`team_hierarchy_holder.cpp`](team_hierarchy_holder.cpp.md) · [`seniority_hierarchy_space.h`](seniority_hierarchy_space.h.md) · [`team_hierarchy_holder_inline.h`](team_hierarchy_holder_inline.h.md)
**Used by** — [`Spectator.cpp`](Spectator.cpp.md) · [`relation_registry_actions.cpp`](relation_registry_actions.cpp.md) · [`seniority_hierarchy_holder.cpp`](seniority_hierarchy_holder.cpp.md) · [`seniority_hierarchy_holder.h`](seniority_hierarchy_holder.h.md) · [`team_hierarchy_holder.cpp`](team_hierarchy_holder.cpp.md) · [`team_hierarchy_holder_inline.h`](team_hierarchy_holder_inline.h.md)
**Tier floor** — T2: a fixed-capacity registry with an upward link.

## Purpose

Declares the surface implemented in
[`team_hierarchy_holder.cpp`](team_hierarchy_holder.cpp.md).

Two things in the declaration are decisions. The capacity is a **compile-time 256**, matching
the width of the squad identifier in the spawn record — not a configurable limit. And the
registry holds an **upward link to its parent**, so that a creature reached through its group
can walk back out to its team without carrying three references itself; that is why the
per-creature coordinates can be three small numbers.

## Exported units

- construction from the parent team registry — see
  [`team_hierarchy_holder_inline.h`](team_hierarchy_holder_inline.h.md).
- `squad(id)` — the squad registry for an identifier, created on first request.
- `team()` — the parent registry.
- `squads()` — the whole slot array, including the empty slots, for anything that needs to
  walk every squad that exists.
