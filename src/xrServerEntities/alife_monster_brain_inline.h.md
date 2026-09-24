# src/xrServerEntities/alife_monster_brain_inline.h

> The monster brain's three field accessors.

**Needs** — [`alife_monster_brain.h`](alife_monster_brain.h.md)
**Used by** — [`alife_monster_brain.h`](alife_monster_brain.h.md)
**Tier floor** — T3.

## Purpose

Carries the trivial accessors split out of
[`alife_monster_brain.h`](alife_monster_brain.h.md): the owning record, the movement
manager, and the get/set pair for the script veto on task selection. The split is a habit
of this codebase, not a decision.

## `owner` / `movement` / `may_choose_tasks`

**Contract** — return the bound record and movement manager (both always present); read and
write the boolean veto. No failure paths.
