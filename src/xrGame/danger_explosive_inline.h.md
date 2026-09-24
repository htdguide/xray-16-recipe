# src/xrGame/danger_explosive_inline.h

> Construction of a tracked-grenade record, and its cheap identity comparison.

**Needs** — [`danger_explosive.h`](danger_explosive.h.md)
**Used by** — [`danger_explosive.h`](danger_explosive.h.md)
**Tier floor** — T3: field assignment

## Purpose

Holds the constructor and the reference comparison for the tracked-grenade record. A
separate file only so the definitions are visible at every call site; a rebuild merges it
away.

## State

`Stateless.` It writes the record described in
[`danger_explosive.cpp`](danger_explosive.cpp.md).

## `CDangerExplosive` construction

**Contract** — takes the explosive, the same entity as a game object, the creature taking
responsibility for reacting, and the time of observation. Assigns all four.

**Invariants** — checks that a present explosive is accompanied by a present game object.
The empty record — no explosive, no object — is the legal "not tracking anything" value, and
is how the record is reset.

## Comparison against an explosive reference

**Contract** — reference identity against the tracked explosive. Yields true for two empty
records, which is consistent: neither is tracking anything.
