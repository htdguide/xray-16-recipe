# src/xrGame/game_cl_mp_snd_messages.h

> The five announcement identifiers common to every multiplayer mode, and the base of a numbering scheme the mode-specific headers extend.

**Needs** — _(none)_
**Used by** — [`game_cl_mp.cpp`](game_cl_mp.cpp.md) · [`game_cl_mp_snd_messages.cpp`](game_cl_mp_snd_messages.cpp.md)
**Tier floor** — T3: one enumeration

## Purpose

The announcement system has one registry shared by every mode, keyed by a numeric identifier.
Each mode contributes its own identifiers from its own header, and **the ranges are laid out
so they cannot collide**: the shared ones occupy the low numbers, deathmatch starts at 100,
team deathmatch at 200, artefact hunt at 300, capture the artefact at 400. That partitioning
is the only thing keeping two modes' announcements apart, since nothing checks it at
registration — so a rebuild must either preserve the ranges or replace the numeric key with
something that cannot collide.

## State

`Stateless.`

## the shared announcement identifiers

**Contract** — five, numbered from zero: headshot, assassin (a kill streak), butcher (a
longer streak), ready (a player declared himself ready), and match started.

**Notes** — the two streak names are the announcer's vocabulary, not the game's: the streak
thresholds that select between them live in the mode's own logic, and the identifiers here
only name the two sounds.
