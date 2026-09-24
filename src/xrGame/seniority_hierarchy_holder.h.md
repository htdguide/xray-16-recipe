# src/xrGame/seniority_hierarchy_holder.h

> Declares the root of the command hierarchy: the registry of teams for one level.

**Needs** — [`seniority_hierarchy_space.h`](seniority_hierarchy_space.h.md) · [`team_hierarchy_holder.h`](team_hierarchy_holder.h.md)
**Used by** — [`Entity.cpp`](Entity.cpp.md) · [`Level.cpp`](Level.cpp.md) · [`Spectator.cpp`](Spectator.cpp.md) · [`ai_monsters_misc.cpp`](ai/ai_monsters_misc.cpp.md) · [`relation_registry_actions.cpp`](relation_registry_actions.cpp.md) · [`seniority_hierarchy_holder.cpp`](seniority_hierarchy_holder.cpp.md) · [`seniority_hierarchy_holder_inline.h`](seniority_hierarchy_holder_inline.h.md)
**Tier floor** — T2: a small owning registry

## Purpose

Declares the surface implemented in
[`seniority_hierarchy_holder.cpp`](seniority_hierarchy_holder.cpp.md) and
[`seniority_hierarchy_holder_inline.h`](seniority_hierarchy_holder_inline.h.md).

## Exported units

- **The class** — owner of up to 64 team holders, addressed by team index.
- **`team(index)`** — the team holder for that index, created on first ask.
- **`teams()`** — the whole registry, read-only, for iteration.
- **Construction and destruction** — fill with empties; delete every team on the way out.

## Notes

The capacity of 64 teams is a compile-time constant with no data-driven source. It caps
the team index a spawn record or a script may use; shipped content uses a handful.
