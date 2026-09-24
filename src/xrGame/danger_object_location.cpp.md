# src/xrGame/danger_object_location.cpp

> A squad warning that follows an object: its position is the object's, it never times out, and it is dropped when that object goes away.

**Needs** — [`danger_object_location.h`](danger_object_location.h.md) · [`GameObject.h`](GameObject.h.md)
**Used by** — reached through its declarations in [`danger_object_location.h`](danger_object_location.h.md); callers name that, not this file.
**Tier floor** — T3: delegation to a bound object

## Purpose

The concrete [danger location](danger_location.h.md) for a *thing* rather than a *place*.
Its three overrides are each one line, but each one changes a base policy, and the changes
are the point of the file.

## State

```text
RECORD DangerObjectLocation EXTENDS DangerLocation
  object : GameObject     # required at construction; never replaced
```

**Invariant** — the bound object is required to exist at construction and is assumed to
exist for every subsequent read. Nothing in this record keeps it alive; the squad brain's
object-destroyed sweep is what guarantees the location is dropped first. That sweep is not
optional — without it this record dereferences a dead object on its next position read.

## `position`

**Contract** — the bound object's current position, read live. This is why the base declares
position as a query rather than a stored field: the warned circle moves with what is
dangerous.

## `useful`

**Contract** — always true. The base's expiry rule is deliberately discarded: a location
about an object is valid exactly as long as the object exists, and object lifetime is
managed by the sweep rather than by a clock. The recorded interval is therefore inert for
this implementation, and the constructor still accepts one.

## Match against a game object

**Contract** — compares entity identifiers. This is what lets the sweep find and remove the
location when the object it names is destroyed, and it is the reason the base's blanket
refusal has to be overridable at all.
