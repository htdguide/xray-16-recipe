# src/xrGame/smart_cover_object_inline.h

> The two enemy-distance thresholds and the cover accessor.

**Needs** — [`smart_cover_object.h`](smart_cover_object.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: accessors

## Purpose

Carries the bodies for [`smart_cover_object.h`](smart_cover_object.h.md).

## `enter_min_enemy_distance` / `exit_min_enemy_distance`

**Contract** — field reads of the two per-placement thresholds, both set from the
[server object](../../GLOSSARY.md) record at spawn. The loophole scoring in
[`smart_cover.cpp`](smart_cover.cpp.md) picks between them by whether the creature is
already in the cover.

## `get_cover`

**Contract** — the placed cover, asserting one exists. A smart cover object spawned
without a description, or spawned on a level with no
[alife](../../GLOSSARY.md) simulation running, has none — so callers must establish that
before asking, and asking anyway is a programming error rather than a recoverable miss.
