# src/xrGame/Level_SLS_Load.cpp

> The level-side hook for restoring a saved game, which is empty because the restore happens entirely on the server side.

**Needs** — [`Level.h`](Level.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T4: it is a name with no body.

## Purpose

The level exposes a symmetric save/load pair so that the engine's level lifecycle can call
either without knowing which side owns the data. Saving does own work (see
[`Level_SLS_Save.cpp`](Level_SLS_Save.cpp.md)); loading does not, because a save is
restored by standing the server up from the snapshot and then letting the normal spawn
path recreate every client object. By the time the level is asked to load, the work is
already done.

A rebuild should keep the hook only if its lifecycle needs the symmetry; otherwise delete
it.

## State

Stateless.

## `load_saved_state`

**Contract** — Accepts a save name and does nothing. Present to complete the lifecycle
surface.
