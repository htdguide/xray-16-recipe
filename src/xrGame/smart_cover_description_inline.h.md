# src/xrGame/smart_cover_description_inline.h

> The three field reads on a smart-cover description.

**Needs** — [`smart_cover_description.h`](smart_cover_description.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: accessors

## Purpose

Carries the bodies for
[`smart_cover_description.h`](smart_cover_description.h.md).

## `table_id` / `loopholes` / `transitions`

**Contract** — plain reads of the authored name, the loophole list and the transition
graph. All three are read on hot paths — the graph by every plan search through a cover —
so none may compute.
