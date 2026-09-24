# src/xrGame/game_base.h

> Declares the game-rules base — the per-player scoreboard record, the team record, the game-state interface every mode implements, and the game clock — implemented in [`game_base.cpp`](game_base.cpp.md).

**Needs** — [`game_base.cpp`](game_base.cpp.md) · [`GameObject.h`](GameObject.h.md) · [`xrServerEntities/game_base_space.h`](../xrServerEntities/game_base_space.h.md) · [`xrServerEntities/alife_space.h`](../xrServerEntities/alife_space.h.md) · [`gametype_chooser.h`](../xrServerEntities/gametype_chooser.h.md) · [`player_account.h`](player_account.h.md) · [`xrEngine/EngineAPI.h`](../xrEngine/EngineAPI.h.md)
**Used by** — [`UIGameCTA.h`](UIGameCTA.h.md) · [`UIPanelsClassFactory.cpp`](UIPanelsClassFactory.cpp.md) · [`UITeamState.h`](UITeamState.h.md) · [`alife_spawn_registry.cpp`](alife_spawn_registry.cpp.md) · [`cta_game_artefact.cpp`](cta_game_artefact.cpp.md) · [`cta_game_artefact.h`](cta_game_artefact.h.md) · [`game_base.cpp`](game_base.cpp.md) · [`game_base_script.cpp`](game_base_script.cpp.md) · [`game_cl_base.cpp`](game_cl_base.cpp.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`game_cl_base_script.cpp`](game_cl_base_script.cpp.md) · [`game_sv_base.cpp`](game_sv_base.cpp.md) · [`game_sv_base.h`](game_sv_base.h.md) · [`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md) · _and 5 more_
**Tier floor** — T2: declares a record whose field order is the network wire format

## Purpose

The declaration shared by every game mode, on both the server and the client side. Substance
is in [`game_base.cpp`](game_base.cpp.md), except for two things the declaration itself
decides and which a rebuild must honour.

**The first is `IGameState`, an interface, and it is substantive**: every mode must supply
it, and the rest of the engine reaches the rules only through it. What it demands:

```text
INTERFACE GameState
  Type()                  -> game_mode_id      # which mode this is
  Phase()                 -> int               # where in the match we are
  Round()                 -> int               # which round; -1 before the first
  StartTime()             -> int               # server time the current phase began
  Create(options)                              # parse the mode's option string
  type_name()             -> text              # for logs and script dispatch
  createPlayerState(account) -> PlayerState    # a mode may extend the player record
  GetStartGameTime()      -> game_time         # in-world clock at match start
  GetGameTime()           -> game_time         # in-world clock now
  Get/SetGameTimeFactor()                      # how fast in-world time runs
  Get/SetEnvironmentGameTime(Factor)           # a second, independent clock
  SetGameTimeFactor(at_time, factor)           # set both origin and rate at once
```

The two clocks are the load-bearing part: the world clock that alife and the mission logic
read, and a second clock that only the weather and sky system reads. They start together and
can be rescaled independently, which is how a script fast-forwards the sky for a cutscene
without also advancing the simulation, or advances the simulation through a night without
a visible sunrise.

**The second is the byte layout.** The whole header is compiled with structure padding
disabled, because the player record is written field by field to the network and its field
*order* is the wire format. A rebuild does not need packed structures — it needs the
serialization order in [`game_base.cpp`](game_base.cpp.md) reproduced exactly.

Exported units:

- `RPoint` — a respawn point: position, orientation, the time it becomes usable again, and a
  reservation (which player holds it and since when) so two players do not materialize
  inside each other. Compares equal to a player identifier when reserved by that player.
- `Bonus_Money_Struct` — one line of the end-of-round money breakdown: an amount, a reason
  code, and how many kills earned it.
- `game_PlayerState` — the per-player record. The single most-touched record in the
  multiplayer code; its fields and invariants are in the implementation twin.
- `game_TeamState` — two numbers: the team's score and how many objectives it currently
  holds.
- `ETeam` — the three team slots: two playing teams and the spectators. Introduced to stop
  the code passing bare small integers around; the spectator slot is a team so that a
  spectator still has a player record in the same table as everyone else.
- `IGameState` — the interface above.
- `game_GameState` — the shared implementation of it: phase, round, start time, the two
  clocks, and the mapping from a mode's name to the class identifier of its server and
  client halves.
