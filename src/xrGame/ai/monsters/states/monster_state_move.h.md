# src/xrGame/ai/monsters/states/monster_state_move.h

> A one-line base for leaves that move: it guarantees the path builder is prepared before the leaf's
> first update.

**Needs** — [`../state.h`](../state.h.md)
**Used by** — [`monster_state_attack_camp_stealout.h`](monster_state_attack_camp_stealout.h.md) · [`monster_state_home_point_rest.h`](monster_state_home_point_rest.h.md) · [`monster_state_smart_terrain_task_graph_walk.h`](monster_state_smart_terrain_task_graph_walk.h.md)
**Tier floor** — T3: one overridden entry step

## Purpose

The smallest file in this directory, and it exists for a real reason. A leaf that hands the path
builder a destination must first tell it that a *new* route is about to be requested — otherwise
the builder keeps serving the previous leaf's route, and the new leaf spends its first updates
following the old one. Several leaves duplicate that call in their own entry step; this base
factors it out for the ones that do not otherwise need an entry step at all.

It is a pure convenience: a rebuild that puts "prepare the path builder" into the leaf base, or
into the path builder's own target-setting call, needs no equivalent of this file. What must
survive is the *rule* — a movement leaf prepares the builder before it first sets a target.

## `CMonsterStateMove`

**Contract** — a state whose entry step runs the base entry step and then prepares the path
builder. Adds no fields and no other behaviour. The optional opaque parameter block that the state
base accepts is forwarded unchanged, so a subclass can still be a parameterized leaf.

**Notes** — the leaves that derive from it in this slice are the wander-home leaf
([`monster_state_home_point_rest.h`](monster_state_home_point_rest.h.md)) and the cross-level walk
to a smart-terrain job
([`monster_state_smart_terrain_task_graph_walk.h`](monster_state_smart_terrain_task_graph_walk.h.md)).
Both are leaves whose only entry work is choosing a destination, so without this base each would
need an entry step for one line.
