# src/xrGame/moving_objects.cpp

> Owns the spatial index of every moving creature: built per level, maintained by register, unregister and reindex.

**Needs** — [`moving_objects.h`](moving_objects.h.md) · [`moving_object.h`](moving_object.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: index lifecycle

## Purpose

The membership half of the obstacle-avoidance system. Everything here is bookkeeping around
one spatial index; the reasoning is in the three sibling twins. It is a separate file
because the lifecycle is what a rebuild must get exactly right and the solver is what a
rebuild may replace.

## State

```text
RECORD MovingObjects
  index               : spatial index of MovingObject, keyed by indexed position
  nearest_static      : list<object>        # scratch: level furniture near a query
  nearest_moving      : list<MovingObject>  # scratch: creatures near a query
  collision_emitters  : list<MovingObject>  # scratch: the solver's work stack
  visited_emitters    : list<MovingObject>  # scratch: what the solver has already swept
  collisions          : list<TimedCollisionAction>   # this frame's findings
  previous_collisions : list<TimedCollisionAction>   # last frame's, for stability
  registered          : set<MovingObject>   # checked builds only
```

**Invariants** — every scratch container is a member rather than a local, so that a solver
pass that runs for every creature every frame allocates nothing. A rebuild in a tier with
cheap allocation may make them locals; a rebuild at this tier should not.

The index is absent until a level is loaded. Registering into an absent index is a
lifecycle error, so creature construction must not outrun level load.

## `on_level_load`

**Contract** — discards any existing index and builds a fresh one over the newly loaded
level. The index's bounds are the navigation mesh's bounding box; its smallest cell is half
the navigation mesh's cell size; it is preallocated for sixteen thousand nodes and sixteen
thousand entries.

**Invariants** — sizing the index from the navigation mesh rather than from the level
geometry is deliberate: only creatures that walk the navigation mesh are ever in this index,
so the mesh's extent is exactly the extent that can be occupied.

**Notes** — the half-cell leaf size means the index resolves at twice the navigation mesh's
resolution. Coarser would put several creatures in one leaf and make every proximity query
scan them; finer would deepen the tree for no gain, because a creature's radius is already
about one cell.

Sixteen thousand of each is a preallocation, not a cap in spirit, but it is one in
practice: the containers are sized once and the level's creature population is expected to
stay well under it. No reason for that exact figure is recoverable beyond it being a round
power-of-two headroom over any shipped level's population.

## `register_object` / `unregister_object`

**Contract** — insert or remove one creature's record in the index. Each asserts, by
creature name, that the record is not already present and that it is present, respectively.
Both require the index to exist.

**Invariants** — this is the local enforcement of the engine-wide rule that no entity is
registered twice and that a destroyed entity is unreferenced before its memory is released.
A rebuild should keep both assertions: a double registration corrupts the index silently
and manifests much later as a creature avoiding a ghost.

## `on_object_move`

**Contract** — reindex a creature after it has moved: remove its record, resample its
position, reinsert it. Asserts the record is registered.

**Notes** — the order is mandatory. Resampling before removing would change the key the
index is about to search by, and the removal would not find the record.

The source marks this as the system's optimization target. The cost is a tree removal and
insertion per moving creature per movement notification; an index supporting in-place
key updates would avoid it.

## `clear`

**Contract** — forgets the previous frame's collision list. Called when the world is torn
down or the solver's history must not carry across a discontinuity.

**Notes** — it clears only the *previous* list, not the current one. The current list is
cleared at the start of every solver pass, so clearing it here would be redundant; carrying
a stale previous list across a level change or a save load would make the solver treat
first-time meetings as repeat encounters and apply the wrong tie-break.
