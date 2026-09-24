# src/Common/Config.hpp

> The compile-time switches that decide which optional behaviours are built into the executable at all.

**Needs** — _(none)_
**Used by** — [`Common.hpp`](Common.hpp.md)
**Tier floor** — T4: a list of build flags. Any language expresses these as build configuration.

## Purpose

A handful of behaviours in this engine are not runtime-configurable: they change the
script-visible surface or the inventory model, so they must be decided before the
executable is built and then held constant, because the shipped Lua scripts and save files
are written against one answer. This file is the single place those answers live.

## State

```text
RECORD BuildFeatures                # all decided at build time, never at runtime
  more_inventory_slots        : bool   # default on: five extra generic carry slots
  game_object_extended_exports: bool   # default on: the wider script facade
  game_object_casting_exports : bool   # default on: script-side downcasts to concrete types
  dead_body_collision         : bool   # default on: corpses keep a collision shape
  actor_before_death_callback : bool   # default off: a hook that runs while the player
                                       #   is dying, before death is final
  log_timing                  : bool   # default off: instrument log writes
```

**Invariants** — the three script-facing switches widen the set of names Lua can see. Turning
one off after scripts have been written against it breaks those scripts, so a rebuild that
wants to load the shipped script set must build with all three on.

## Notes

`dead_body_collision` restores a behaviour the original engine removed: corpses are solid
rather than walk-through. It is a gameplay decision, not an optimization, and both answers
are self-consistent.

The player-death callback is off by default because it extends the window in which the
player entity is alive but committed to dying; anything that assumes "alive implies
interactive" has to be re-checked when it is on.
