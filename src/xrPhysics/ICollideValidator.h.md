# src/xrPhysics/ICollideValidator.h

> Hands out collision-group identifiers to the game layer without exposing the
> filtering rules.

**Needs** — [`PHCollideValidator.h`](PHCollideValidator.h.md)
**Used by** — [`PHCollideValidator.cpp`](PHCollideValidator.cpp.md) · [`PHCollideValidator.h`](PHCollideValidator.h.md) · [`PhysicsShell.h`](PhysicsShell.h.md)
**Tier floor** — T3: one counter.

## Purpose

Objects that must never collide with each other — the parts of one ragdoll, a vehicle and
its wheels, an object and the thing it was spawned inside — are put in a shared *group*, and
members of the same group are filtered out of collision before any contact is computed. The
game layer needs to mint a fresh group, and nothing else. This one-line header is that.

The rules themselves are in [`PHCollideValidator.h`](PHCollideValidator.h.md).

## `RegisterGroup`

**Contract** — returns a new group identifier, distinct from every one previously returned
since the last world reset. Identifiers are dense and monotonically increasing; there is no
release. A rebuild that expects thousands of short-lived groups per level would need to
reclaim them, but the shipped content does not: groups are minted per spawned composite
object and the counter resets with the level.
