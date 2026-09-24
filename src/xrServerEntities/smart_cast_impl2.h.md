# src/xrServerEntities/smart_cast_impl2.h

> Defining mode for the cast table: each entry becomes a one-line cast body that calls the facet method the entry names.

**Needs** — [`smart_cast.h`](smart_cast.h.md) · [`smart_cast.cpp`](smart_cast.cpp.md)
**Used by** — [`smart_cast.h`](smart_cast.h.md)
**Tier floor** — T1.

## Purpose

The counterpart to [`smart_cast_impl0.h`](smart_cast_impl0.h.md). Read in this mode, a table
entry stops being a declaration and becomes the cast itself:

```text
FUNCTION cast_to<Target>(source: Source) -> optional<Target>
  RETURN source.<facet method named by the entry>()
```

That is the entire scheme. The facet method is a virtual call the source's own class
implements — returning itself when it is a Target, nothing when it is not — so the cast
costs one indirect call and no type search.

In this mode the table is **not** rebuilt; only one unit reads it this way and it needs
bodies, not rows.

## Notes

**Two entries are written out by hand** rather than generated, and both are interesting.

The facade's downcast from its interface to its single implementation is a **plain static
conversion**: the interface has exactly one implementing type, so no check is performed at
all. If a second implementation ever appeared, every such cast would silently produce a
wrong pointer.

The sound-sensing facet is hand-written because its type lives in a nested namespace that
the table's entry form cannot name.
