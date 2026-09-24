# src/xrGame/ef_storage.h

> Declares the registry of every evaluation function and the shared parameter block they all read their inputs from; implemented in [`ef_storage.cpp`](ef_storage.cpp.md), [`ef_storage_inline.h`](ef_storage_inline.h.md) and [`ef_storage_script.cpp`](ef_storage_script.cpp.md).

**Needs** — [`ef_base.h`](ef_base.h.md) · [`ef_storage_inline.h`](ef_storage_inline.h.md)
**Used by** — [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) · [`ai_monsters_misc.cpp`](ai/ai_monsters_misc.cpp.md) · [`ai_stalker_fire.cpp`](ai/stalker/ai_stalker_fire.cpp.md) · [`ai_space.cpp`](ai_space.cpp.md) · [`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md) · [`alife_monster_abstract.cpp`](alife_monster_abstract.cpp.md) · [`alife_update_manager.cpp`](alife_update_manager.cpp.md) · [`ef_base.h`](ef_base.h.md) · [`ef_pattern.cpp`](ef_pattern.cpp.md) · [`ef_primary.cpp`](ef_primary.cpp.md) · [`ef_storage.cpp`](ef_storage.cpp.md) · [`ef_storage_inline.h`](ef_storage_inline.h.md) · [`ef_storage_script.cpp`](ef_storage_script.cpp.md) · [`enemy_manager.cpp`](enemy_manager.cpp.md) · _and 1 more_
**Tier floor** — T3: a declaration plus two small generic constructions

## Purpose

Declares `CEF_Storage`, the one instance that owns every evaluation function in the game,
plus the two generic pieces that make the family work: the parameter block and the
enemy-perspective wrapper. Those two are declared only here and are substance.

## The parameter block

**Contract** — four slots, filled by the caller before asking any function for a value: the
creature doing the evaluating (*member*), the creature being evaluated against (*enemy*), and
one item belonging to each.

```text
RECORD Params<Creature, Item>
  member      : optional<Creature>
  enemy       : optional<Creature>
  member_item : optional<Item>
  enemy_item  : optional<Item>
```

**Invariant** — there are **two** instances of this block, of different types: one holding
*client objects* (live entities and game objects) and one holding *server objects* (alife
records). A function reads whichever is filled. This is how one set of functions serves both
the detailed simulation, where creatures are live, and the alife simulation, where they are
records. The convention is: the client block is checked first, and being empty there means
the alife block is in use. Every single function in
[`ef_primary.cpp`](ef_primary.cpp.md) begins with that test.

**Invariant** — exactly one block may be filled at a time. Switching is by clearing the
other; see `alife_evaluation` in [`ef_storage_inline.h`](ef_storage_inline.h.md). Filling
both leaves the client block winning silently.

**Invariant** — the block is shared mutable state. The whole evaluation system is therefore
single-threaded, non-reentrant, and unsafe to call from inside another evaluation — except
via the one wrapper below, which saves and restores it.

## The enemy-perspective wrapper

**Contract** — takes any *personal* function and produces the same function answered about
the **enemy** instead of about the member.

```text
FUNCTION enemy_view_of(F).value()
  saved = the currently-filled parameter block
  block.member      = block.enemy
  block.member_item = block.enemy_item
  result = F.value()
  block = saved
  RETURN result
```

**Invariants** — the swap is temporary and the block is restored in full afterwards, which
makes this the only safe re-entrant path through the system. It is how "enemy health",
"enemy creature type", "enemy weapon type", "enemy eye range" and "enemy maximum health" are
defined: they are not separate functions, they are the personal ones seen from the other
side. A rebuild that passes the subject as an argument deletes this wrapper entirely and
should.

**Notes** — the five wrapped functions shadow the names of five separately declared classes
in the same header. The shadowed declarations are unused; the type aliases win. That is
confusing in the source and a rebuild should keep only one of each name.

## `CEF_Storage`

Exported units:

- construction and destruction — build every function and destroy them; see
  [`ef_storage.cpp`](ef_storage.cpp.md).
- `m_fpaBaseFunctions` — a fixed array of 128 slots holding every function **by index**. The
  index is the identity by which trained data refers to a function; the gaps in it are
  deliberate and are described in [`ef_storage.cpp`](ef_storage.cpp.md).
- named handles for each function — roughly thirty primary ones and twenty-four trained ones,
  so that engine code can reach a specific function without a lookup.
- `function(name)` — lookup by name, for scripts.
- `alife_evaluation(bool)` — choose which parameter block is live.
- `non_alife()` / `alife()` — the two blocks.
- `script_register` — the Lua surface; see
  [`ef_storage_script.cpp`](ef_storage_script.cpp.md).
