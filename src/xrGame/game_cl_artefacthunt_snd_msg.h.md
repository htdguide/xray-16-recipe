# src/xrGame/game_cl_artefacthunt_snd_msg.h

> Artefact hunt's twenty announcement identifiers: every combination of which team, which event, and whose perspective.

**Needs** — _(none)_
**Used by** — [`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md) · [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md)
**Tier floor** — T3: one enumeration

## Purpose

One enumeration continuing the shared numbering scheme described in
[`game_cl_mp_snd_messages.h`](game_cl_mp_snd_messages.h.md). Artefact hunt's block begins at
300 and is by far the largest, because its announcements are three-dimensional.

## State

`Stateless.`

## the artefact-hunt announcements

**Contract** — two mode-wide events (a new objective appeared, the objective was lost), then
three events — the objective reached a base, the objective was taken, the objective was
returned — each stated **six** ways: per owning team, and per listener perspective.

The three perspectives are the decision worth naming. For each event and team the set is:

- **plain** — spoken to a listener with no stake, or the default recording;
- **`_R`** — spoken to a listener on the team the event is *about*, the "our" rendering;
- **`_ENEMY`** — spoken to a listener on the other team, the "their" rendering.

So "team one took the artefact" is a different line depending on whether you are on team one,
on team two, or neither. The mode picks the identifier by comparing the listener's team with
the event's team; the enumeration exists to make all nine combinations addressable.

**Invariants** — the three perspectives for one (event, team) pair are adjacent but the
grouping order is **not uniform across the three events**: the base-arrival group lists both
teams' plain identifiers first and then both teams' other perspectives, while the take and
return groups list a team's three perspectives together. Any arithmetic over these
identifiers must therefore be written per group; a rebuild indexing them by a computed offset
will be wrong. Address them by name.
