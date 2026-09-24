# src/xrGame/game_type.cpp

> Answers, for any code anywhere in the game layer, whether it is running the authoritative side, the local side, or a single-player session.

**Needs** — [`game_type.h`](game_type.h.md) · [`Level.h`](Level.h.md)
**Used by** — reached through its declarations in [`game_type.h`](game_type.h.md); callers name that, not this file.
**Tier floor** — T3: three predicates over global session state

## Purpose

Entity code constantly branches on whether it owns the authoritative copy of a decision or
is merely mirroring one: damage is applied on the server, prediction runs on the client,
save/load only exists on the server. These three predicates are the only sanctioned way to
ask, and they are deliberately cheap and total — they answer `false` rather than failing
when no level is loaded, so that code running during startup, shutdown or a level
transition does not have to guard every call site.

## State

`Stateless.` All three read global session objects.

## `OnServer`

**Contract** — true when a level is loaded *and* that level holds the authoritative side.
False when no level is loaded. Never fails.

## `OnClient`

**Contract** — true when a level is loaded *and* that level holds a local, mirroring side.
False when no level is loaded. Never fails.

**Notes** — the two are not exclusive. In a single-player session, and on a listen server,
both are true at once: one process holds both the server object and the client object of
every entity. Code that means "only once per entity" must therefore test `OnServer`, not
`not OnClient`.

## `IsGameTypeSingle`

**Contract** — true when the persistent game's selected game type is the single-player one.
Independent of whether a level is currently loaded, because the game type is chosen before
the level is.

**Notes** — the distinction from `OnServer` matters: single player is always server-side,
but a multiplayer listen host is also server-side and must not take the single-player
branches (alife, saves, level change by script).
