# src/xrGame/agent_location_manager_inline.h

> Construction, reset, and looking a danger up by the object that caused it.

**Needs** — [`agent_location_manager.h`](agent_location_manager.h.md) · [`danger_location.h`](danger_location.h.md)
**Used by** — [`agent_location_manager.h`](agent_location_manager.h.md)
**Tier floor** — T3: list lookup

## Purpose

The small half of the location manager. The scoring and arbitration are in
[`agent_location_manager.cpp`](agent_location_manager.cpp.md).

## State

Adds nothing.

## Construction

**Contract** — binds the squad brain this manager belongs to, permanently; requires it to
be present.

## `location` (by causing object)

**Contract** — returns the danger caused by a given object, or nothing. A linear scan over
the danger list; the list holds a handful of entries.

**Notes** — the match is performed by an equality between a danger record and an object,
which is the same operation used to drop a departing object's dangers. That one predicate
serving both lookup and removal is why the removal helper is defined here rather than with
the removal code.

## `locations` / `clear`

**Contract** — a read-only view of every active danger, and a full reset for a squad being
torn down.

## `object`

**Contract** — the squad brain; requires the binding.
