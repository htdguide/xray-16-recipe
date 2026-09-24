# src/xrGame/stalker_movement_restriction_inline.h

> The three-operation filter a stalker hands to the cover search — admit, weight,
> claim — all delegated to the squad's shared location bookkeeping.

**Needs** — [`stalker_movement_restriction.h`](stalker_movement_restriction.h.md) · [`agent_location_manager.h`](agent_location_manager.h.md) · [`cover_point.h`](cover_point.h.md)
**Used by** — [`stalker_movement_restriction.h`](stalker_movement_restriction.h.md)
**Tier floor** — T2.

## Purpose

The cover search is generic over a filter; this is the stalker's. The important content is
not the three one-line bodies but the *shape* they impose on any cover search in a
rebuild: a search is not "find the best point", it is "find the best point **this**
creature is allowed to have, then mark it as taken so the next creature in the squad gets
a different one".

## State

```text
RECORD StalkerCoverFilter
  object              : StalkerRef
  agent_manager       : AgentManagerRef      # the squad's shared coordinator
  use_enemy_info      : bool                 # whether admission accounts for where the
                                             # squad believes the enemy is
  notify_agent_manager: bool                 # whether choosing actually claims the point
```

**Invariants** — the agent manager is captured at construction from the stalker, so the
filter is only valid while that stalker is a member of that squad.

## Construction

**Contract** — binds to a stalker, reads its squad's agent manager, and records the two
flags. The claiming flag defaults to true; passing false yields a *probing* filter that
evaluates which covers would be acceptable without taking one, which is what a planner
does when it is costing a hypothetical action.

## Admission

**Contract** — true when the squad's location bookkeeping considers this cover point
suitable for this stalker. Suitability is a squad-level judgement — it accounts for points
already claimed by squadmates and, when enemy information is enabled, for whether the
point is exposed to where the squad believes the enemy is. Side-effect free.

## `weight`

**Contract** — the danger of the point for this stalker, as rated by the squad's shared
model. The cover search minimises this, so lower is safer. Side-effect free.

## `finalize`

**Contract** — called once on the point the search settles on. Marks it as taken by this
stalker in the squad's bookkeeping, **unless** the filter was built in probing mode. This
is the only mutating operation of the three and it is what keeps a squad from stacking up
in one doorway.
