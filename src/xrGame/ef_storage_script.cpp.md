# src/xrGame/ef_storage_script.cpp

> Exports the evaluation-function catalogue to Lua as a single overloaded `evaluate` call taking a function name and up to four objects.

**Needs** — [`ef_storage.h`](ef_storage.h.md) · [`ef_base.h`](ef_base.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [`ai_space.h`](ai_space.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: argument marshalling

## Purpose

Exposes the evaluation system to scripts. The exported surface is deliberately tiny — one
free function returning the catalogue, and one method with eight overloads — because the
catalogue itself is addressed by *name string*, which keeps the binding from having to know
about fifty function classes.

## State

Stateless; it fills the catalogue's shared parameter block per call.

## `ef_storage`

**Contract** — the global catalogue instance, reached through the AI subsystem's service
locator. Exported to Lua as a free function.

## `evaluate`

**Contract** — eight overloads in two families of four. Each takes the catalogue, a function
name, and one to four objects; the missing trailing arguments are filled with nothing. The
two families differ only in the object type — **game objects** (the script-visible facade over
live entities) or **server objects** (alife records) — and that choice is what selects which
parameter block the evaluation reads.

```text
FUNCTION evaluate(catalogue, name, o0, o1, o2, o3) -> real
  catalogue.select_block(this family)         # clears the other block
  f = catalogue.function(name)
  IF f is absent THEN log a script error ; RETURN 0
  block.member = o0 as a living entity
  IF o0 was given but is not a living entity THEN log a script error ; RETURN 0
  block.enemy  = o1 as a living entity
  IF o1 was given but is not a living entity THEN log a script error ; RETURN 0
  block.member_item = o2
  block.enemy_item  = o3
  RETURN f.value()
```

**Invariants** — the argument positions are fixed and meaningful: the evaluating creature,
the creature it is evaluating against, an item of the first, an item of the second. A script
passing them in another order gets a wrong answer rather than an error, because only the
first two are type-checked.

**Invariants** — the first two arguments must be living entities and the last two need not
be. That asymmetry is the parameter block's own shape.

**Invariants** — every failure path returns zero and logs, rather than raising. A script
naming a function that does not exist gets a plausible-looking number. This is the script
surface's general policy — see [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer),
where the shipping build compiles this layer without exceptions — but it is worth flagging
because zero is a meaningful value for most of these functions.

**Notes** — the shorter overloads pass nothing for the trailing objects, so the same slot
that a two-argument call leaves empty a four-argument call fills. Since the block is shared
and the mode switch clears it, an unfilled slot is reliably empty rather than stale. That is
the only thing making the overloads safe.

**Notes** — the error messages for both type checks read "not inherited from a schedulable
alife object" in both families, which is wrong for the game-object family — it checks for a
living entity. A rebuild should say what it checked.
