# src/xrGame/alife_level_registry.h

> The subset of the offline world that lies on the currently loaded level: a mutation-safe table the simulation walks every frame while its own callbacks add and remove entries.

**Needs** — [`alife_level_registry_inline.h`](alife_level_registry_inline.h.md) · [`safe_map_iterator.h`](safe_map_iterator.h.md) · [`xrServerEntities/xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`xrAICore/Navigation/game_graph.h`](../xrAICore/Navigation/game_graph.h.md) · [`ai_debug.h`](ai_debug.h.md)
**Used by** — [`alife_graph_registry.cpp`](alife_graph_registry.cpp.md) · [`alife_graph_registry.h`](alife_graph_registry.h.md) · [`alife_level_registry_inline.h`](alife_level_registry_inline.h.md) · [`movement_manager_game.cpp`](movement_manager_game.cpp.md)
**Tier floor** — T3: a keyed table with a resumable iteration.

## Purpose

The simulation's per-frame work is bounded by a time budget, so it cannot visit every object
on the level every frame; it visits as many as fit and resumes where it left off. And the
visits themselves promote objects online, demote them offline and destroy them, which means
the table is being modified while it is being walked. This type is the level subset with
both properties: it is a keyed table that survives mutation during iteration and remembers
its position between frames.

Most of that behaviour lives in the mutation-safe table it extends; what this type adds is
the *level filter* and the identity of the level.

## State

```text
RECORD LevelRegistry            # extends a mutation-safe map<entity id, object>
  level_id : int                # which level this subset is for

# Invariant: every object in the table has a graph vertex belonging to this level.
#   The add operation enforces it by refusing anything else, silently.
# Invariant: the table holds references; the object registry owns the objects.
```

## `add`

**Contract** — Admits an object only when its graph vertex belongs to this level; anything
else is silently ignored. That silent filter is what lets the graph registry offer it every
object without checking first (see
[`alife_graph_registry.cpp`](alife_graph_registry.cpp.md)).

## `remove`

**Contract** — Removes an object by entity identifier. Missing is a hard error by default,
downgradable by a flag — used when an object is leaving a level and the caller does not
know whether it was ever on this one.

## `update`

**Contract** — Runs a visitor over as many objects as the time budget allows, resuming from
where the previous frame stopped, and taking a flag saying whether the next frame should
start over from the beginning instead. Safe against the visitor adding, removing or
destroying entries.

**Notes** — A build switch exists that forces every frame to restart from the beginning —
that is, to attempt the whole level every frame rather than resuming. It is off. With it
on, the simulation is exhaustive and the time budget becomes a hard truncation rather than
a rotation; a rebuild should keep the rotating form, since exhaustive updating starves the
objects at the end of the table whenever the budget is tight.

## `object`

**Contract** — Looks an object up by entity identifier. Absent is a hard error by default,
downgradable by the same flag convention every alife registry uses.

## `level_id`

**Contract** — The level this subset belongs to.
