# src/xrGame/ai/monsters/ai_monster_squad_manager_inline.h

> The lazy global accessor for the one creature-pack registry.

**Needs** — [`ai_monster_squad_manager.h`](ai_monster_squad_manager.h.md)
**Used by** — [`ai_monster_squad_manager.h`](ai_monster_squad_manager.h.md)
**Tier floor** — T3: one pointer and a create-on-first-use

## Purpose

There is exactly one creature-pack registry per process, reached by every creature through
a free function. The function creates the registry on first call and never destroys it.

## `monster_squad`

**Contract** — returns the one registry, constructing it if it does not exist yet. Never
fails. Not thread-safe — two simultaneous first calls would both construct — which is safe
only because every caller is on the simulation thread.

**Notes** — this is one of the engine's service-locator reaches described in
[`SYSTEM-REQUIREMENTS.md`](../../../../SYSTEM-REQUIREMENTS.md#7-build-order). What is
actually being reached for is *the level's creature-pack coordination*, which is level
scope, not process scope: the registry is never torn down between levels, so packs from a
previous level persist as empty entries keyed by the same triples. That is invisible
because the triples are reused and the entries are empty, but a rebuild that owns this from
the level rather than from a global is strictly more correct.
