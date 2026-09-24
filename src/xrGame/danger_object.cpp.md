# src/xrGame/danger_object.cpp

> Nothing: the danger record is entirely declared in its header.

**Needs** — [`danger_object.h`](danger_object.h.md)
**Used by** — reached through its declarations in [`danger_object.h`](danger_object.h.md); callers name that, not this file.
**Tier floor** — T3: nothing to implement

## Purpose

Exists only to give the record's destructor a single home, a C++ linkage concern with no
counterpart in a rebuild. The substance is in [`danger_object.h`](danger_object.h.md).

## State

Stateless.
