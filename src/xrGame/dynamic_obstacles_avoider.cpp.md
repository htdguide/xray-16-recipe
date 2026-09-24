# src/xrGame/dynamic_obstacles_avoider.cpp

> Avoiding other creatures rather than walls: take the navigation cells the traffic registry says are claimed by somebody else, and stand still entirely when the registry has told this creature to wait.

**Needs** — [`dynamic_obstacles_avoider.h`](dynamic_obstacles_avoider.h.md) · [`static_obstacles_avoider.h`](static_obstacles_avoider.h.md) · [`moving_objects.h`](moving_objects.h.md) · [`moving_object.h`](moving_object.h.md) · [`ai_space.h`](ai_space.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — reached through its declarations in [`dynamic_obstacles_avoider.h`](dynamic_obstacles_avoider.h.md); callers name that, not this file.
**Tier floor** — T3: set operations over navigation vertices

## Purpose

Two creatures walking toward each other down a corridor must not both keep going and must
not both stop. The engine solves that with a global registry of moving objects that arbitrates
between them — see [`moving_objects.cpp`](moving_objects.cpp.md) — which hands each creature
two things: a verdict (move or wait) and the set of navigation cells other creatures have
claimed.

This file is the small adapter that feeds those two answers into the avoidance machinery the
[static avoider](static_obstacles_avoider.cpp.md) already implements. All the path-detouring
work is inherited; only the *source* of obstacles and one early exit are new.

## State

None of its own. It reuses the base's two obstacle sets and its current-iteration scratch
set.

## `query`

**Contract** — replaces the base's obstacle gathering. Asks the registry to arbitrate for
this creature, then takes ownership of the resulting set of claimed navigation cells.

```text
FUNCTION query()
  registry.arbitrate(this creature's moving-object record)
  swap the base's current-iteration set with the record's reported claimed set
```

**Invariants** — the set is *swapped* out of the record rather than copied. That is not an
optimization detail worth preserving as such, but the consequence is: the record's set is
emptied by the query, so a second query in the same frame reports nothing. The registry
refills it on the next arbitration.

## `movement_enabled`

**Contract** — the registry's verdict for this creature, as a boolean: waiting means no,
moving means yes. Any other verdict is a hard failure, so adding a traffic state without
handling it here is caught immediately.

## `process_query`

**Contract** — narrows the two obstacle sets to the cells still claimed this iteration, then
runs the base's avoidance. Returns the base's answer, or success immediately when the
creature has been told to wait.

```text
FUNCTION process_query(change_path_state) -> bool
  IF NOT movement_enabled THEN RETURN true      # I am waiting; nothing to avoid
  inactive_obstacles = inactive_obstacles ∩ current_iteration
  active_obstacles   = active_obstacles   ∩ current_iteration
  RETURN base.process_query(change_path_state)
```

**Invariants** — intersecting rather than replacing is what gives the sets their memory. A
cell stays in the obstacle set only while it is *still* claimed; a creature that has walked
on releases its cells and they fall out of every other creature's sets on the next pass. This
is the entire ageing policy for dynamic obstacles, and it is why obstacles do not need
timestamps.

**Invariants** — a waiting creature reports success without touching the sets at all. It has
no path to detour and its stale obstacle sets are irrelevant until it moves again. Running
the avoidance while waiting would make the creature repeatedly rebuild a path it is not
allowed to walk, which is the deadlock the registry exists to prevent.
