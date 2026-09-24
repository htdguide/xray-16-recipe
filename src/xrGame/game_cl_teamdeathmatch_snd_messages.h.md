# src/xrGame/game_cl_teamdeathmatch_snd_messages.h

> Team deathmatch's fifteen announcement identifiers, occupying the two-hundred block.

**Needs** — _(none)_
**Used by** — [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md) · [`game_cl_teamdeathmatch.cpp`](game_cl_teamdeathmatch.cpp.md)
**Tier floor** — T3: one enumeration

## Purpose

One enumeration continuing the shared numbering scheme described in
[`game_cl_mp_snd_messages.h`](game_cl_mp_snd_messages.h.md). Team deathmatch's block begins
at 200, and every identifier in it is stated twice, once per team.

## State

`Stateless.`

## the team-deathmatch announcements

**Contract** — team one wins, team two wins, the teams are level; team one leads, team two
leads; then two contiguous runs of five rank announcements, one run per team.

**Invariants** — the per-team duplication is the pattern the whole team-mode announcement set
follows: the announcer speaks in the listener's own frame, so a listener on team two hears a
different recording of "we are ahead" than a listener on team one, and the mode selects the
identifier by comparing the listener's team with the subject's. Each run of five ranks is
contiguous and indexed by adding the rank, exactly as in
[`game_cl_deathmatch_snd_messages.h`](game_cl_deathmatch_snd_messages.h.md), so the two runs
must stay five long and adjacent.
