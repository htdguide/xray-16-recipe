# src/xrServerEntities/alife_movement_manager_holder.h

> The state an offline entity needs in order to be *somewhere on the game graph and going somewhere else*, and the three things its owner must be able to answer.

**Needs** — [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md)
**Used by** — [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md)
**Tier floor** — T2.

## Purpose

An offline entity does not have a position; it has an *edge in progress*. This interface is
the shape of that: the two graph vertices it is between, how fast it moves, how far along it
is, and which terrain kinds it may cross. Anything that can be moved by the alife simulation
implements it, which in practice means every creature record.

It is an interface rather than a record because the movement manager has to reach back into
its owner for two things it cannot know — the owner's record, and the notification that the
owner has changed level — and inheriting the fields is how this codebase expresses "these
fields belong to whatever is being moved".

## State

```text
RECORD MovementState
  next_vertex         : GameGraphVertex   # where this entity is heading
  previous_vertex     : GameGraphVertex   # where it came from
  travel_speed        : real              # on the game graph, between levels
  level_travel_speed  : real              # within one level
  current_speed       : real
  distance_from_point : real              # progress along the current edge, from previous
  distance_to_point   : real              # remaining, to next
  terrain_mask        : list<int>         # which terrain kinds this entity may traverse
```

**Invariants**

- The position of an offline entity is `(previous_vertex, next_vertex, distance_from_point)`
  and nothing else. It is only converted to a world coordinate when the entity goes online.
- `distance_from_point + distance_to_point` is the length of the current edge; the pair is
  kept rather than one value plus a lookup because the edge's length comes from the graph
  and the simulation advances the split.
- Two speeds exist because crossing a level boundary and walking within a level are charged
  differently: the graph edge between levels is long and abstract, the edges inside one are
  not.

## The three demands on an implementor

**`on_location_change`** — tell me when this entity's graph vertex changes, so that the
registries keyed by level (which entities are on which map) can be updated. Declared
`const` because the notification is conceptually observation, not mutation, of the holder.

**`record()`** — give me the entity record you belong to, mutable and immutable. The
movement manager needs it to read the entity's terrain preferences and to write its new
vertex.

## Notes

**The terrain mask is per entity, not per class.** It comes from the creature's
configuration and decides which game-graph edges are passable for it — which is why a
mutant does not wander into a town and a stalker does not cross a swamp. It is stored here,
with the movement state, because the pathfinder that consumes it is the one in the movement
manager.
