# src/xrGame/stalker_movement_manager_obstacles_inline.h

> The one accessor of the obstacle layer: its obstacle-aware restrictor.

**Needs** — [`stalker_movement_manager_obstacles.h`](stalker_movement_manager_obstacles.h.md)
**Used by** — [`stalker_movement_manager_obstacles.h`](stalker_movement_manager_obstacles.h.md)
**Tier floor** — T4: one accessor

## Purpose

A separate file because the declaration includes it at its end, which is how C++ defines a
member after the class is complete. Nothing here is a decision.

## `restricted_object`

**Contract** — the layer's obstacle-aware restrictor, which is the same object the base
manager holds as its generic restrictor, typed so that the obstacle-specific operations are
reachable. It always exists once configuration has been loaded; asking before that is a
caller error.
