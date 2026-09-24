# src/xrGame/ui/UIActorMenuDeadBodySearch.cpp

> Looting: a corpse, a living companion or a container on one side, the actor's bag on the
> other, plus the one primitive every ownership transfer in the chapter is built from.

**Needs** — [`UIActorMenu.h`](UIActorMenu.h.md) · [`UIWeightBar.h`](UIWeightBar.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UICellItemFactory.h`](UICellItemFactory.h.md) · [`UICharacterInfo.h`](UICharacterInfo.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: list orchestration plus packed event construction

## Purpose

Three different things are looted through the same screen — a dead body, a living partner who
has agreed to share, and a container in the world — and the differences between them are
small and local. This file also defines `move_item_from_to`, which is the primitive
*every* cross-inventory transfer in the chapter goes through, including trade.

## `move_item_from_to` — the ownership primitive

**Contract** — Move an item from one entity's inventory to another's. Free function, used by
every file in the actor-menu family.

```text
FUNCTION move_item_from_to(from_id, to_id, item_id)
  # A transfer is expressed as a sell followed by a buy, even when no
  # trade is happening and no money moves. That pair is the only
  # ownership move the entity event system understands, so every
  # transfer -- looting, giving, storing -- is spelled this way.
  submit event(kind: trade sell, to: from_id, payload: item_id)
  submit event(kind: trade buy,  to: to_id,   payload: item_id)
```

**Invariants** — The order is not negotiable: the item must leave before it arrives, or the
authoritative side sees it in two inventories at once. Neither event carries a price; the
money side of a real trade is settled separately.

## `move_item_check`

**Contract** — The same, with an optional weight test against the receiver's carry limit
first. Returns whether the move happened. The test is `current + item >= maximum`, so an
exact-fit item is refused — a deliberate margin, not an off-by-one.

Every call site in this file passes *false* for the weight test, so take-all currently ignores
carry weight. That is the original's behaviour and is visible in gameplay.

## `InitDeadBodySearchMode`

**Contract** — Entry. Shows the loot window, both bag lists, the partner weight bar and the
take-all button; shows the partner's character panel only when there *is* a partner (a
container has none).

```text
FUNCTION init_loot_mode()
  show the loot widgets
  IF the partner panel's parent is a separate icon frame THEN hide that frame
      # one layout dialect frames the portrait; a container has no portrait to frame

  refill the actor side from the actor's inventory

  IF there is a partner
    items = partner.available_items(include equipped: only if the partner is alive)
    refresh the partner weight bar
  ELSE
    mark the container in use
    items = container.available_items()

  sort items by descending footprint
  fill the loot list

  # Looting a dead human -- not an animal, not a container -- also
  # transfers everything they knew.
  IF the partner is a dead, non-animal character
    FOR EACH info portion the partner knew
      submit event(kind: information transfer, to: actor, payload: that portion)
    clear the partner's known information

  refresh the loot-side weight readout
```

**Invariants** — Three decisions here are game rules, not presentation:

- **A living partner's equipped items are not shown.** The `is_alive` flag becomes "include
  equipped items"; a corpse yields everything, a companion yields only their spare kit.
- **Looting a corpse transfers their knowledge.** Every information portion the dead character
  held is given to the actor and then erased from the corpse, so a body can only be read once.
  Animals and containers are exempt — they knew nothing.
- **A container is marked in use** while the screen is open, which is what stops two actors
  looting it simultaneously in multiplayer. Exit must clear the mark; it does.

## `DeInitDeadBodySearchMode`

**Contract** — Hide everything and release the container's in-use mark. Runs even when entry
never ran, so every step is guarded.

## `ToDeadBodyBag`

**Contract** — Put an item *into* the corpse or container — the direction the player uses to
store things.

```text
FUNCTION to_loot_bag(cell, at_cursor) -> bool
  IF there is a partner AND the partner refuses deposits THEN RETURN false
  IF there is a container AND the container refuses deposits THEN RETURN false
  IF the item is a quest item THEN RETURN false

  move the cell from its list into the loot list
      (at the cursor's cell when the gesture was a drop, else the next free cell)

  move_item_from_to(actor, partner or container, the item)
  refresh the loot-side weight readout
  RETURN true
```

**Invariants** — **Quest items can never be stored.** Putting a quest item in a container the
player might never return to would make a quest unfinishable, so the refusal is absolute
rather than a warning.

Both the partner and the container carry their own "will you accept deposits" flag, set by the
game; a looted corpse typically will, a scripted stash may not.

## `TakeAllFromPartner` / `StoreAllToPartner`

**Contract** — The two bulk moves, wired to the take-all button and to a keyboard shortcut.
Each walks a list, moves **every stack child first and then the head**, and finally clears the
list wholesale rather than removing cells one at a time.

```text
FUNCTION take_all_from_partner()
  IF there is no partner THEN delegate to the container variant; RETURN
  FOR EACH cell IN the loot list
    FOR EACH child IN cell.stack
      move_item_check(child.item, partner, actor, weight check: off)
    move_item_check(cell.item, partner, actor, weight check: off)
  clear the loot list
```

**Invariants** — Children before heads, always. A head's payload can be swapped with a
child's during a split (see [`UICellItem.cpp`](UICellItem.cpp.md)), so taking the head first
would leave the list holding a cell whose item has already moved.

The list is cleared in bulk rather than incrementally because the acquire notifications coming
back from the game will rebuild the actor's side; incremental removal would fight them.

## `TakeAllFromInventoryBox` / `StoreAllToInventoryBox`

**Contract** — The same two sweeps against a container instead of a partner, addressing the
container by its entity identifier. They exist separately only because a container is not an
inventory owner and cannot be weight-checked; a rebuild that unifies the two abstractions
collapses these four operations into two.
