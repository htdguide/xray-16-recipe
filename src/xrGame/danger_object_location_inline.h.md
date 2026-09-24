# src/xrGame/danger_object_location_inline.h

> Construction of the object-bound danger location.

**Needs** — [`danger_object_location.h`](danger_object_location.h.md)
**Used by** — [`danger_object_location.h`](danger_object_location.h.md)
**Tier floor** — T3: field assignment

## Purpose

The constructor body for [`danger_object_location.cpp`](danger_object_location.cpp.md)'s
type, split out for inlining.

## State

Adds nothing.

## Construction

**Contract** — binds the object (required to be present) and copies the timestamp, the
interval, the radius and the squad mask into the base record. The mask defaults to
all-bits-set, meaning *every* squad member is warned; a caller that means one member must
say so explicitly. Defaulting to warning everyone is the safe direction: a missed warning
is a death, a spurious one is a detour.

**Notes** — the interval is stored even though this implementation's `useful` ignores it.
A rebuild can drop the parameter here.
