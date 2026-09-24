# src/xrGame/mt_config.h

> The ten switches that decide which of the game's per-frame workloads may be handed to a worker thread.

**Needs** — _(none)_
**Used by** — [`Level_Bullet_Manager.cpp`](Level_Bullet_Manager.cpp.md) · [`alife_update_manager.cpp`](alife_update_manager.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md) · [`movement_manager.cpp`](movement_manager.cpp.md) · [`movement_manager_game.cpp`](movement_manager_game.cpp.md) · [`movement_manager_level.cpp`](movement_manager_level.cpp.md) · [`movement_manager_patrol.cpp`](movement_manager_patrol.cpp.md)
**Tier floor** — T3: a flag vocabulary over a shared setting

## Purpose

One global flag word plus the names of its bits. Every expensive per-frame subsystem asks
this word whether it is allowed to defer its work, so that the whole engine's threading
policy is one runtime-settable value rather than a compile-time decision scattered through
ten files.

## State

```text
GLOBAL mt_config : int (32-bit, a set of flags)   # settable at run time

ENUM Workload                 # bit positions within that word
  level_path      = bit 0     # navigation-mesh search for one creature
  detail_path     = bit 1     # smoothing a level path into travel points
  object_handler  = bit 2     # the planner that decides how a creature uses its weapon
  sound_player    = bit 3     # selecting and starting a creature's sounds
  ai_vision       = bit 4     # the visibility raycasts that feed perception
  bullets         = bit 5     # advancing projectiles and resolving their hits
  lua_gc          = bit 6     # stepping the script virtual machine's collector
  level_sounds    = bit 7     # the level's ambient sound scheduling
  alife           = bit 8     # the off-screen world simulation
  map             = bit 9     # rebuilding the map screen's marker set
```

**Invariants** — each bit is *permission*, not a command. A subsystem that finds its bit set
may still run its work inline when the caller demanded an immediate result, when the object
concerned is being destroyed, or when the work is too small to be worth queueing. The
movement manager's rule is the model: defer only when the bit is set, no at-once build was
demanded, and the object is not dying.

The ten workloads are the ten places in the frame where the engine found enough work to be
worth moving off the critical path. Their identity is the information here — a rebuild
choosing its own concurrency strategy still wants to know that these ten, and not others,
are where the time goes.

**Notes** — the script collector appearing on this list is the one entry that is not
parallelism in the usual sense: the script virtual machine is single-threaded, and the
switch controls whether its collection is stepped off the frame's critical section rather
than genuinely concurrently. A rebuild whose script tier collects on its own schedule can
drop that bit.

Nothing here declares *how many* workers exist or how work is queued; this file is only the
policy vocabulary. A rebuild may keep the word exactly as it is, since its whole value is
that it is one place.
