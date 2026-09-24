# src/xrGame/game_base_kill_type.h

> The three vocabularies a multiplayer kill is classified by: what killed you, what was special about it, and what it was worth.

**Needs** — _(none)_
**Used by** — [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`game_cl_base_weapon_usage_statistic.h`](game_cl_base_weapon_usage_statistic.h.md) · [`game_sv_deathmatch.cpp`](game_sv_deathmatch.cpp.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md) · [`game_sv_mp.h`](game_sv_mp.h.md)
**Tier floor** — T3: three enumerations

## Purpose

A shared header holding nothing but three enumerations, separated from the game rules so
that the client, the server, the score tables and the announcement sounds all classify a
death the same way. The split is otherwise arbitrary — these could live in
[`game_base.h`](game_base.h.md) — but keeping them alone means the modules that only need
the vocabulary do not pull in the rules.

## State

`Stateless.`

## `KILL_TYPE`

**Contract** — how the victim died. Three causes: a hit, bleeding out, radiation. The
distinction exists because only a hit has a killer with a weapon; the other two are
attributed to whoever last damaged the victim, and the announcement and statistics paths
branch on it.

## `SPECIAL_KILL_TYPE`

**Contract** — the decoration on a kill that earns an announcement and, in some modes, a
bonus. Eight values: none, headshot, backstab, knife kill, killed-with-a-scanner, a kill in
a row (a streak), a rank promotion, and an eye shot.

**Notes** — rank promotion and kill-in-a-row are not properties of a single kill at all;
they are states of the killer that the same channel announces, reusing this enumeration as
the announcement identifier rather than the kill's own classification. A rebuild that keeps
kill classification and announcement selection separate will find the two sets are not the
same set.

## `KILL_RES`

**Contract** — what the kill was *worth*, which is the only one of the three the scoring
rules read. Six values: nothing, a suicide, a teammate, a teammate at a moment that matters
to the mode, a rival, a rival at a moment that matters to the mode.

**Invariants** — the "critical" pair is mode-specific: in the artefact modes it means the
victim was carrying the objective. A mode with no objective never produces the critical
values, and the scoring table must still define entries for them.
