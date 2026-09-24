# src/xrGame/alife_monster_movement_manager_inline.h

> The accessors of the offline movement arbiter, split out of the header as a C++ habit.

**Needs** — [`alife_monster_movement_manager.h`](alife_monster_movement_manager.h.md)
**Used by** — [`alife_monster_movement_manager.h`](alife_monster_movement_manager.h.md)
**Tier floor** — T3: field reads and one field write

## Purpose

Bodies for the always-inlined accessors of `CALifeMonsterMovementManager`. A rebuild
folds these into the type. Substance is in
[`alife_monster_movement_manager.cpp`](alife_monster_movement_manager.cpp.md).

## Accessors

**Contract** — `object`, `detail` and `patrol` return the owner and the two owned
sub-managers by reference; `path_type` reads and writes the movement mode.

**Invariants** — the owner and both sub-managers are present for the manager's whole
lifetime. That is asserted at every access rather than checked once, which is a statement
about the intended shape: these are *references*, not optional links, and a rebuild
should express them as such and delete the checks.
