# src/xrGame/eatable_item_object.cpp

> The concrete consumable entity: two behaviours joined into one object, with an explicit ordering at every lifecycle point.

**Needs** — [`eatable_item_object.h`](eatable_item_object.h.md) · [`eatable_item.h`](eatable_item.h.md) · [`physic_item.h`](physic_item.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: dispatch ordering

## Purpose

A consumable in the world is two things at once: a physical object with a collision shell
that can be dropped, thrown and shot, and an inventory item with uses and effects. This file
joins them. It contains almost no logic — it is a list of lifecycle points and, for each one,
the order in which the two halves are told about it.

**That order is the entire content of the file, and it is load-bearing.** A rebuild composing
the two behaviours differently still has to answer the same twenty questions.

## State

None of its own. Both halves carry theirs.

## The orderings

Read as a table of "who first".

| Lifecycle point | First | Then | Why |
|---|---|---|---|
| construct | consumable | physical | the consumable half caches a reference to the physical one, so it must run while the object is still being assembled |
| `Load` | physical | consumable | the consumable's weight tunables are derived from the weight the physical half loaded |
| `net_Spawn` | physical | consumable | the world object must exist before the use counter is seeded against it; the physical half's answer is the one returned |
| `net_Destroy` | consumable | physical | drop the inventory-side state before the world object goes |
| `reinit` | consumable | physical | |
| `reload` | physical | consumable | mirrors `Load` |
| `save` / `load` | physical | consumable | fixes the field order in the save stream; reversing it silently corrupts every existing save |
| entering a container | physical | consumable | |
| leaving a container, before | consumable | physical | the consumable half hides and disables a spent item *before* the physical half places it in the world |
| leaving a container, after | consumable | physical | the consumable half may destroy a spent item; the extra visibility clearing here is a belt-and-braces repeat of the "before" step |
| `UpdateCL`, `OnEvent`, `Hit`, `renderable_Render` | physical | consumable | the per-frame and event paths all run the world half first |

**Invariants** — the save and load orders match each other and must not change. The stream is
positional.

**Invariants** — the leaving-a-container pair is the only place this file adds behaviour of
its own: after the transition it re-hides a spent item. The comment in the source says why —
an item that has been used up and is being dropped as part of its own destruction must not
appear on the ground for the frame before it goes.

## The single-delegate members

**Contract** — several hooks go to exactly one half rather than both, and which half is the
decision:

- `net_Import` / `net_Export`, `make_Interpolation` and the four physics
  correction-prediction hooks go to the **consumable** half only. The network representation
  of a consumable is its inventory state, not its physics; a dropped tin is not worth
  reconciling.
- `activate_physic_shell` goes to the consumable half, while `on_activate_physic_shell` goes
  to the **physical** half's activation. The two names are nearly identical and they route to
  different places; this is a trap for anyone reading quickly, and a rebuild should name them
  for what they do.
- `NeedToDestroyObject` goes to the inventory-item base directly, skipping both halves'
  overrides.

## `Useful` · `ef_weapon_type`

**Contract** — `Useful` is the consumable half's answer verbatim, republished so the object's
own interface carries it. `ef_weapon_type` is zero: a consumable has no place in the
weapon-evaluation scheme the AI uses to choose what to shoot with.

**Notes** — the file also carries a commented-out earlier signature of the hit path, from
before damage was bundled into a single record. It documents that a hit used to be seven
loose arguments; the bundling is the reason the two-line forwarding here is readable at all.
