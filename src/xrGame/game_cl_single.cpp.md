# src/xrGame/game_cl_single.cpp

> Single-player rules: build the single-player screen, and take the clock from the alife simulation instead of from the match.

**Needs** — [`game_cl_single.h`](game_cl_single.h.md) · [`game_cl_base.h`](game_cl_base.h.md) · [`UIGameSP.h`](UIGameSP.h.md) · [`Actor.h`](Actor.h.md) · [`clsid_game.h`](../xrServerEntities/clsid_game.h.md) · [`ai_space.h`](ai_space.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_time_manager.h`](alife_time_manager.h.md) · [`Level.h`](Level.h.md)
**Used by** — [`game_cl_single.h`](game_cl_single.h.md)
**Tier floor** — T3: delegation and a factory call

## Purpose

The thinnest of the modes. Single player has no scores, no teams, no voting and no remote
players, so almost every hook the base declares is left alone. What it does decide is
**which clock is authoritative**, and that decision is the whole file.

## State

```text
GLOBAL difficulty : DifficultyLevel   # process-wide, defaults to the second level
GLOBAL difficulty_names : map<text, DifficultyLevel>   # four names for the console
```

**Notes** — the default difficulty is the second of four, not the first: the game is meant
to be met at its middle setting, and the novice level exists as an opt-out.

## `createGameUI`

**Contract** — instantiates the single-player screen through the class factory by its class
identifier, loads it, binds it back to this rules object, and initialises its three layers.
Fails hard if the factory returns something of the wrong kind.

**Notes** — the three initialisation calls with indices 0, 1 and 2 are the screen's layers
(the head-up display, the message area and the map indicators); the screen's own twin
defines them. Going through the factory rather than constructing directly is what lets a mod
substitute a different screen class by class identifier.

## the clock overrides

**Contract** — every clock accessor asks the same question first: is the alife simulation up?

```text
FUNCTION game_time()
  IF alife exists AND is initialized THEN RETURN alife.time_manager.game_time()
  ELSE RETURN base.game_time()          # the linear map over server real time
```

The same shape covers the start time, the rate, and both environment-clock accessors — and
crucially, **the environment clock is the game clock** in single player: both getters return
the alife time, so the sky and the simulation cannot diverge. The independent second clock
that multiplayer has does not exist here.

Setting the rate does *not* go through the base at all: it is forwarded to the **server-side**
rules object, which owns the authoritative clock even in one process. The environment setters
forward the same way while alife is up, and fall back to the base's own clock before it is.

**Invariants** — the fallback is not a nicety. The rules object is constructed before the
level loads and therefore before alife exists, and the loading screen reads the clock. Every
accessor must answer during that window.

Routing writes through the server side and reads through alife is the correct expression of
"the server is authoritative": in single player both live in one process, so the round trip
is immediate, but the direction of authority is the same as in multiplayer.

## `IsServerControlHits`

**Contract** — always true. With one process there is no untrusted client, so the local
arbitration is the server's arbitration.

## `getTeamSection`

**Contract** — none. There are no teams.

## `OnDifficultyChanged`

**Contract** — forwards to the actor, which re-reads the damage and condition multipliers
its difficulty selects. The difficulty is a console variable, so it can change mid-game.
