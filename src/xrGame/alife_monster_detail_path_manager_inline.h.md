# src/xrGame/alife_monster_detail_path_manager_inline.h

> The trivial accessors of the offline detail-path mover, split out of the header purely as a C++ habit.

**Needs** — [`alife_monster_detail_path_manager.h`](alife_monster_detail_path_manager.h.md)
**Used by** — [`alife_monster_detail_path_manager.h`](alife_monster_detail_path_manager.h.md)
**Tier floor** — T3: field reads and one guarded field write

## Purpose

This file exists because the original separates a class's declaration from its
always-inlined bodies. Nothing here is a decision a rebuild must repeat: it is the
accessor set of `CALifeMonsterDetailPathManager`, and a rebuild should fold it into the
type itself. Substance lives in
[`alife_monster_detail_path_manager.cpp`](alife_monster_detail_path_manager.cpp.md).

## Accessors

**Contract** — `object` returns the owning movement-manager holder. `speed` reads and
writes the travel speed. `path` returns the inverted vertex path. `walked_distance`
returns how far along the current edge the creature has travelled.

**Invariants** — worth carrying into a rebuild, because they are the only content here:

- the owner reference is never absent for the lifetime of the manager;
- the speed is always a finite real, both when set and when read — an infinite or
  not-a-number speed would silently teleport the creature, so it is rejected at the
  setter rather than at the integrator;
- `walked_distance` is only meaningful while a path of at least two vertices exists; a
  single-vertex path means "already there" and there is no edge to be partway along.
