# src/xrGame/game_cl_capturetheartefact_snd_msg.h

> Capture the artefact's two announcement identifiers, occupying the four-hundred block.

**Needs** — _(none)_
**Used by** — [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md)
**Tier floor** — T3: one enumeration

## Purpose

One enumeration continuing the shared numbering scheme described in
[`game_cl_mp_snd_messages.h`](game_cl_mp_snd_messages.h.md). Capture the artefact's block
begins at 400.

## State

`Stateless.`

## the capture-the-artefact announcements

**Contract** — two identifiers: team one's artefact was returned, team two's artefact was
returned.

**Notes** — the mode is far richer than two announcements suggest. Everything else it says —
captures, scores, the round countdown — it says through the on-screen caption system in
[`game_cl_capture_the_artefact_captions_manager.cpp`](game_cl_capture_the_artefact_captions_manager.cpp.md)
and through the inherited team-deathmatch set, not through recorded voice. Only the return
event got recordings of its own.
