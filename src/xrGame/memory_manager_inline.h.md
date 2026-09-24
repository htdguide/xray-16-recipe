# src/xrGame/memory_manager_inline.h

> The accessors onto a creature's six memory sub-managers, and the enumeration of remembered objects that count as enemies.

**Needs** — [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md)
**Used by** — [`memory_manager.h`](memory_manager.h.md)
**Tier floor** — T3: field reads and one filtered walk

## Purpose

Carries the memory manager's accessors and one small algorithm out of the declaration.

## State

`Stateless.`

## `visual` · `sound` · `hit` · `enemy` · `item` · `danger` · `object` · `stalker`

**Contract** — hand out the six sub-managers, the owning creature, and the creature in its
stalker form. Every one asserts the target exists. The stalker accessor is only valid on a
stalker; a monster asking for it is a caller bug.

## `fill_enemies`

**Contract** — calls a caller-supplied action once for every remembered object that is a
living entity the creature's enemy judgement considers hostile. Skips muted entries. Two
forms: one over a given remembered-object list, one over the creature's whole memory.

**Invariant** — the whole-memory form walks the **visual** memory only. The sound and hit
memories are deliberately excluded — the calls are present and commented out — so a creature
enumerating its enemies this way lists only enemies it has *seen*, not ones it has merely
heard or been shot by. That is what the callers want: the enumeration feeds squad-level target
distribution, and a squad should not be assigned to shoot at something nobody can see.

```text
FUNCTION fill_enemies(objects, action)
  FOR EACH entry IN objects
    SKIP IF the entry is muted
    SKIP IF the entry's object is not a living entity
    SKIP IF the enemy judgement does not consider it hostile
    action(it)
```
