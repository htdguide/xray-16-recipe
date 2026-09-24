# src/xrGame/cover_point_inline.h

> The cover point's constructor, accessors and approximate equality.

**Needs** — [`cover_point.h`](cover_point.h.md)
**Used by** — [`cover_point.h`](cover_point.h.md)
**Tier floor** — T3: field access

## Purpose

Supplies the bodies for [`cover_point.h`](cover_point.h.md), split out so they inline — which
matters here because cover points are compared inside per-creature evaluation loops. A rebuild
has no second file.

## State

`Stateless.`

## the operations

**Contract** — construction from a position and a navigation vertex, with the smart-cover flag
cleared; readers for both; and equality by approximate position only. The reasoning behind the
approximate comparison is in [`cover_point.h`](cover_point.h.md) and is the only thing in this
file a rebuild must preserve.
