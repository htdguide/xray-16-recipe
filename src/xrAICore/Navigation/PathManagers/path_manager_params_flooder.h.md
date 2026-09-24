# src/xrAICore/Navigation/PathManagers/path_manager_params_flooder.h

> The request for a flood fill — everything reachable within a radius — expressed as the base limits with a radius-shaped default.

**Needs** — [`path_manager_params.h`](path_manager_params.h.md)
**Used by** — [`path_manager.h`](path_manager.h.md) · [`path_manager_level_flooder.h`](path_manager_level_flooder.h.md) · [`path_manager_level_flooder_inline.h`](path_manager_level_flooder_inline.h.md)
**Tier floor** — T2: a defaults record.

## Purpose

Extends the base limits with nothing but different defaults, and by being a *distinct type*
selects the flood-fill policy. That is its whole job: the type is the request.

## State

```text
RECORD FloodRequest EXTENDS SearchLimits
  max_range              : real  # reinterpreted as a RADIUS in world units from the start
                                 #   vertex. Default 6000, which is far larger than any
                                 #   level and therefore means "unbounded" — every real
                                 #   caller passes its own.
  max_iteration_count    : int   # default: no limit
  max_visited_node_count : int   # default 65530
```

**Invariants** — the visited budget sits just under the search engine's fixed 65536-vertex
pool, with a smaller margin than the base record's 65500. Nothing depends on the exact
difference; both are "as many as the pool holds".

**Notes** — the record carries one unused integer field. It has no reader anywhere and no
discoverable purpose; a rebuild should drop it.

Reinterpreting the range field rather than adding a radius field is a piece of inherited
awkwardness: the flood policy also replaces the limit test so that the base record's
"estimated total cost" reading never applies. A rebuild should name the field for what the
policy means by it.
