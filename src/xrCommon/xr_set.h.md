# src/xrCommon/xr_set.h

> The engine's ordered key collection, allocating through the engine allocator, with the ordering left open.

**Needs** — [`xr_allocator.h`](xr_allocator.h.md) · [`predicates.h`](predicates.h.md)
**Used by** — [`_stl_extensions.h`](../xrCore/_stl_extensions.h.md) · [`xrCore.h`](../xrCore/xrCore.h.md)
**Tier floor** — T3: an allocator injection point, plus one decision about ordering.

## Purpose

Exists to inject an allocator, and to fix a default ordering. Two forms are named: one
rejecting duplicate keys, one admitting them.

## `xr_set` / `xr_multiset`

**Contract** — a collection of keys iterated in key order, parameterised by the comparison
that defines that order; default is the key type's natural "less than".

**Invariants**

```text
# Membership is decided by the ORDERING, not by equality.
#   Two keys are the same key exactly when neither orders before the other. With
#   the case-folding text ordering (predicates.h) this makes the collection
#   case-insensitive, which is the point at every site that uses it - resource
#   names in the shipped game data are authored with inconsistent case.

# Iteration order is the ordering, and is observable.
#   Same caveat as the ordered table: results get written out in this order.
```

## Declaration shorthands

**Contract** — two macro forms declaring a named collection type with its iterator, one
with the ordering defaulted and one with it given explicitly. Only the second carries
information. A rebuild deletes both.
