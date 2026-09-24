# src/xrGame/game_cl_deathmatch_snd_messages.h

> Deathmatch's eleven announcement identifiers, occupying the hundred-block reserved for it.

**Needs** — _(none)_
**Used by** — [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md) · [`game_cl_deathmatch.cpp`](game_cl_deathmatch.cpp.md)
**Tier floor** — T3: one enumeration

## Purpose

One enumeration continuing the shared numbering scheme described in
[`game_cl_mp_snd_messages.h`](game_cl_mp_snd_messages.h.md). Deathmatch's block begins at
100.

## State

`Stateless.`

## the deathmatch announcements

**Contract** — you won; five rank announcements, one per rank a player can be promoted to;
and five countdown announcements, one per remaining second from one to five.

**Invariants** — the two runs of five are **contiguous and ordered**, which is load-bearing:
the code that announces a rank adds the rank number to the first rank identifier, and the
countdown adds the seconds remaining to the first countdown identifier. Reordering or
interleaving them breaks both. A rebuild that keys announcements by name rather than by
arithmetic must still keep the five-rank and five-second sets complete.
