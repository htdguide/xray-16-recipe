# src/xrGame/game_sv_capture_the_artefact_process_event.cpp

> Capture the artefact's four extra server events: suicide, purchase finished, and entering or leaving a team base — with the base identifier shifted to zero-based.

**Needs** — [`game_sv_capture_the_artefact.h`](game_sv_capture_the_artefact.h.md) · [`xrServer.h`](xrServer.h.md) · [`xrMessages.h`](../xrServerEntities/xrMessages.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a dispatch on an event type

## Purpose

One dispatch function in a file of its own; the split is a compilation artifact and a rebuild
folds it into
[`game_sv_capture_the_artefact.cpp`](game_sv_capture_the_artefact.cpp.md).

## State

`Stateless.`

## `OnEvent`

**Contract** — handles four events and passes the rest to the base: the suicide and the
purchase, exactly as deathmatch does (see
[`game_sv_deathmatch_process_event.cpp`](game_sv_deathmatch_process_event.cpp.md), including
the same asymmetry about whose identity is trusted), and the two team-base events.

**Invariants** — the base identifier is **decremented by one** before use, because the level
editor numbers the green team's base volume 1 and the blue team's 2, while the mode indexes
its teams from zero. The source marks the adjustment as unwelcome, and it is: the same field
in
[`game_sv_artefacthunt_process_event.cpp`](game_sv_artefacthunt_process_event.cpp.md) is not
adjusted at all. The level data is the authority and it is one-based; a rebuild should
convert once where the level is loaded and let every mode see zero-based team indices.
