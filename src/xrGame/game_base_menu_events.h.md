# src/xrGame/game_base_menu_events.h

> The three requests a player can make of the game rules from the in-match menu.

**Needs** — _(none)_
**Used by** — [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md) · [`game_cl_capturetheartefact_buywnd.cpp`](game_cl_capturetheartefact_buywnd.cpp.md) · [`game_cl_teamdeathmatch.cpp`](game_cl_teamdeathmatch.cpp.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md)
**Tier floor** — T3: one enumeration

## Purpose

One enumeration, alone in a header so that the menu screens and the rules modules agree on
the identifiers without either including the other.

## State

`Stateless.`

## `GAME_MENU_EVENTS`

**Contract** — the outcome of the in-match menu, reported back to the game-rules object:
become a spectator, change team, change skin. Each is a *request*, not a command — the rules
decide whether it is legal now (a team change may be refused when teams would become
unbalanced, a skin change may be deferred to the next respawn), which is why the menu
reports an intent rather than performing the change.
