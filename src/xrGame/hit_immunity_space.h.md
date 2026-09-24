# src/xrGame/hit_immunity_space.h

> Names the fixed-length table type that holds one damage multiplier per damage type.

**Needs** — [`xrServerEntities/alife_space.h`](../xrServerEntities/alife_space.h.md) · [`xrCore/FixedVector.h`](../xrCore/FixedVector.h.md)
**Used by** — [`hit_immunity.cpp`](hit_immunity.cpp.md) · [`hit_immunity.h`](hit_immunity.h.md)
**Tier floor** — T2: a type name; the fixed capacity is the only decision

## Purpose

The immunity table is indexed by damage type and is exactly as long as there are damage
types. Naming the type in its own file lets code that only stores such a table avoid
depending on the behaviour in [`hit_immunity.h`](hit_immunity.h.md).

## State

```text
# a fixed-capacity list of multipliers, one slot per damage type, never resized
HitTypeTable : list<real>    # length == number of damage types, always
```

**Invariants** — the capacity is bounded at compile time by the damage-type count. That is
the load-bearing decision: an immunity table is small, is embedded by value inside every
creature, outfit and artefact that has one, and is indexed on every hit, so it must not be a
separate allocation.
