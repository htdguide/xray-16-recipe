# src/xrGame/moving_objects_inline.h

> One accessor: the collisions the avoidance solver resolved last frame.

**Needs** — [`moving_objects.h`](moving_objects.h.md)
**Used by** — [`moving_objects.h`](moving_objects.h.md)
**Tier floor** — T3: field access

## Purpose

A single inline accessor in its own file, following this directory's three-header
convention. The split is arbitrary; a rebuild folds it into the type.

## State

`Stateless.`

## `collisions`

**Contract** — hands out the *previous* frame's resolved collision list, not the current
one. The name says `collisions` and the field it returns is the history.

**Notes** — the naming is worth flagging because the distinction is load-bearing. The
solver compares each new pairwise collision against this list to decide whether the two
creatures have met before, and gives a repeat encounter a different tie-break than a first
one. A rebuild that returns the live list from an accessor with this name will make the
solver compare the current pass against itself and lose all decision stability.
