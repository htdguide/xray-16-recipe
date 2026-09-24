# src/xrGame/alife_level_registry_inline.h

> The level subset's operations: a filter on the way in, a pass-through on the way out.

**Needs** — [`alife_level_registry.h`](alife_level_registry.h.md) · [`ai_space.h`](ai_space.h.md)
**Used by** — [`alife_level_registry.h`](alife_level_registry.h.md)
**Tier floor** — T3: table operations.

## Purpose

Holds the definitions of the operations declared in
[`alife_level_registry.h`](alife_level_registry.h.md), which is where the contracts are
written. The only substance is the filter.

## `add`

```text
FUNCTION add(object)
  IF the level of object's graph vertex is not this registry's level
    RETURN                    # silently: callers offer every object
  insert (object.id, object)
```

**Invariants** — The filter reads the object's *graph* vertex, not its level vertex or its
position, so an object whose graph vertex has not yet been synchronized with its position
is filtered against the stale answer. Every caller synchronizes first; the ordering is a
real requirement rather than an accident.

## `remove`, `object`, `update`, `level_id`

**Contract** — Pass-throughs to the mutation-safe table, in the registry's usual shape: a
hard error by default, downgradable by a flag. `update` forwards the visitor and the
restart flag unchanged.
