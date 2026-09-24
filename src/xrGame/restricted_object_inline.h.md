# src/xrGame/restricted_object_inline.h

> Construction and the three trivial reads.

**Needs** — [`restricted_object.h`](restricted_object.h.md)
**Used by** — [`restricted_object.h`](restricted_object.h.md)
**Tier floor** — T3: field reads

## Purpose

Nothing is decided here. It exists because the type's substantive methods live in a
compilation unit that the creature classes do not include.

## Construction

**Contract** — binds the mixin to its creature, which must exist. Note what is *not*
initialized: neither the applied flag, the removed flag nor the actual flag is set here.
They are set at spawn instead, which makes spawn — not construction — the point after which
this type may be used.

## `applied` · `object` · `actual`

**Contract** — plain reads: whether a temporary border is installed, the creature this mixin
belongs to, and whether the pathfinder's cached cost model is still valid for this
creature's current restriction set.

## `initialize` (debug builds only)

**Contract** — removes a leftover border if one is installed. A reset hook for the debug
path that re-creates a creature's brain without going through destruction; there is no
release-build equivalent, so a rebuild that keeps a reset path must decide whether the
border survives it.
