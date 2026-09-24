# src/xrGame/ai/monsters/bloodsucker/bloodsucker_vampire_approach.h

> Declares the run-in that opens a feed.

**Needs** — [`state.h`](../state.h.md) · [`bloodsucker_vampire_approach_inline.h`](bloodsucker_vampire_approach_inline.h.md)
**Used by** — [`bloodsucker_vampire_approach_inline.h`](bloodsucker_vampire_approach_inline.h.md) · [`bloodsucker_vampire_inline.h`](bloodsucker_vampire_inline.h.md)
**Tier floor** — T3: a declaration over the shared state contract

## Purpose

Declares the surface implemented in [`bloodsucker_vampire_approach_inline.h`](bloodsucker_vampire_approach_inline.h.md). A leaf state with no fields: entry primes the path builder, and every update re-issues the same movement request at the enemy's current cell.

## `VampireApproachState`

A leaf state overriding entry and update only. It declares no start or completion test, so the enclosing vampire tree decides when the approach is over.
