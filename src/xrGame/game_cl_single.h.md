# src/xrGame/game_cl_single.h

> Declares the single-player game rules, and the four difficulty levels, implemented in [`game_cl_single.cpp`](game_cl_single.cpp.md).

**Needs** — [`game_cl_single.cpp`](game_cl_single.cpp.md) · [`game_cl_base.h`](game_cl_base.h.md)
**Used by** — [`ShootingObject.cpp`](ShootingObject.cpp.md) · [`console_commands.cpp`](console_commands.cpp.md) · [`game_cl_single.cpp`](game_cl_single.cpp.md)
**Tier floor** — T3: a declaration plus one enumeration

## Purpose

Declares the client-side rules for single player. Substance is in
[`game_cl_single.cpp`](game_cl_single.cpp.md).

The declaration is mostly a list of *clock* overrides, which tells the reader what single
player actually changes: almost nothing about rules, and everything about where time comes
from.

Exported units:

- `game_cl_Single` — the mode. Builds the single-player screen, reports that hits are always
  server-arbitrated (which in one process means locally arbitrated and trusted), supplies no
  team section, and redirects every clock accessor.
- `OnDifficultyChanged` — forwards a difficulty change to the actor, which re-reads its
  tuned values.
- `ESingleGameDifficulty` — four levels, ordered from easiest to hardest. The value is an
  index, so the order is load-bearing: configuration tables are indexed by it.
- `g_SingleGameDifficulty` — the current level, a process-wide setting rather than per-save
  state.
- `difficulty_type_token` — the mapping between the four levels and the names the console
  and the configuration files use. A token table like this is the engine's standard way of
  giving an enumeration text names for the console; the names are frozen because user
  settings store them.
