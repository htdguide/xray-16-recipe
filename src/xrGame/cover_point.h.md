# src/xrGame/cover_point.h

> One place a creature can stand to be hidden: a position, the navigation vertex it sits on, and whether it is an authored cover or a computed one.

**Needs** — [`cover_point_inline.h`](cover_point_inline.h.md) · [`cover_point_script.cpp`](cover_point_script.cpp.md)
**Used by** — [`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md) · [`agent_location_manager.cpp`](agent_location_manager.cpp.md) · [`agent_member_manager.cpp`](agent_member_manager.cpp.md) · [`base_monster_path.cpp`](ai/monsters/basemonster/base_monster_path.cpp.md) · [`bloodsucker_attack_state_hide_inline.h`](ai/monsters/bloodsucker/bloodsucker_attack_state_hide_inline.h.md) · [`bloodsucker_predator_inline.h`](ai/monsters/bloodsucker/bloodsucker_predator_inline.h.md) · [`bloodsucker_predator_lite_inline.h`](ai/monsters/bloodsucker/bloodsucker_predator_lite_inline.h.md) · [`control_path_builder_base_path.cpp`](ai/monsters/control_path_builder_base_path.cpp.md) · [`corpse_cover.cpp`](ai/monsters/corpse_cover.cpp.md) · [`monster_cover_manager.cpp`](ai/monsters/monster_cover_manager.cpp.md) · [`monster_home.cpp`](ai/monsters/monster_home.cpp.md) · [`psy_dog_state_psy_attack_hide_inline.h`](ai/monsters/pseudodog/psy_dog_state_psy_attack_hide_inline.h.md) · [`monster_state_attack_camp_inline.h`](ai/monsters/states/monster_state_attack_camp_inline.h.md) · [`monster_state_home_point_attack_inline.h`](ai/monsters/states/monster_state_home_point_attack_inline.h.md) · _and 29 more_
**Tier floor** — T2: a packed three-word record that exists in bulk and is compared in hot loops

## Purpose

The unit of the cover system. Everything that answers "where can I stand where he cannot see
me" — the evaluators in [`cover_evaluators.cpp`](cover_evaluators.cpp.md), the per-creature
search in [`cover_manager.cpp`](cover_manager.cpp.md) — produces and consumes these.

A cover point is deliberately *two* things at once: a free-floating world position, and a
vertex of the level's navigation mesh. The position is what the creature actually stands at
and what the visibility test uses; the vertex is what the pathfinder can reach and what the
shipped per-vertex cover values are indexed by. Storing only one of them would make either
the reasoning or the movement impossible.

## State

```text
RECORD CCoverPoint
  position        : vector
  level_vertex_id : int (31-bit)   # packed
  is_smart_cover  : bool (1-bit)   # packed alongside
```

**Invariants**

- The vertex identifier is stored in **31 bits**, one short of a word, with the smart-cover
  flag occupying the last. That caps a level's navigation mesh at just over two billion
  vertices, which is far beyond anything shippable, and the packing exists because cover
  points are created in bulk per creature per evaluation and the record's size is the cost.
  A rebuild is free to use two fields; it should know that it is trading memory in a hot path
  for clarity.
- The position must lie on or just above the named vertex. Nothing enforces it; every
  producer is responsible.
- The flag distinguishes a point derived from the level's own per-vertex cover data from one
  belonging to an authored *smart cover* — a hand-placed piece of cover with animations
  attached, where a creature does not merely stand but leans, crouches and fires from
  specific poses. Consumers branch on it, so it is not decoration.

## `operator==`

**Contract** — two cover points are the same if their **positions** are approximately equal.
Neither the vertex nor the flag participates.

**Invariants** — this is the file's one genuinely load-bearing decision and it is easy to get
wrong. Equality is approximate because cover points are recomputed each evaluation and a
recomputed point for the same spot differs in the last bits; an exact comparison would make
"am I already at my cover" always false and creatures would shuffle in place forever.
Excluding the vertex is the same argument from the other side: one position can be attributed
to either of two adjacent vertices depending on which producer made it.

**Notes** — the consequence is that equality is not transitive. Three points spaced just under
the tolerance apart compare equal pairwise but the outer two may not. Nothing here sorts or
uniquifies cover points, so it does not bite; a rebuild that does must not use this comparison
as an ordering.

## construction and accessors

**Contract** — built from a position and a vertex, with the smart-cover flag false; readers
for the position and the vertex. The flag has no accessor at this level and is read directly
by the systems that set it.
