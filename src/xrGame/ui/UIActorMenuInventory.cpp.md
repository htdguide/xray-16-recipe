# src/xrGame/ui/UIActorMenuInventory.cpp

> The rules for moving an item between slot, belt, bag and quick slot — and the one decision
> that governs the whole chapter: **a UI action never calls the game, it emits a game event.**

**Needs** — [`UIActorMenu.h`](UIActorMenu.h.md) · [`UICellItemFactory.h`](UICellItemFactory.h.md) · [`UICellCustomItems.h`](UICellCustomItems.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UIDragDropReferenceList.h`](UIDragDropReferenceList.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`UIItemInfo.h`](UIItemInfo.h.md) · [`UIOutfitInfo.h`](UIOutfitInfo.h.md) · [`UIMainIngameWnd.h`](UIMainIngameWnd.h.md) · [`../../xrUICore/PropertiesBox/UIPropertiesBox.h`](../../xrUICore/PropertiesBox/UIPropertiesBox.h.md) · [Seam: Script virtual machine](../../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Networking transport](../../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: inventory orchestration plus packed event construction

## Purpose

This is the largest file in the chapter and the one a rebuilder must read before any other,
because it settles the question every inventory UI has to answer: *what actually happens when
the player drags a magazine into a slot?*

The answer is: the screen moves its own widgets immediately, **and separately emits a game
event** describing the intent. The event travels the same path a networked client's request
would, through the entity event system, and is applied by the authoritative side. The screen
does not wait for the result and does not reconcile — in single player both sides are the same
process and the effect is immediate; in multiplayer the screen is optimistic.

That decision is what makes this chapter portable across single player and multiplayer, and
it is the single most important thing on this page.

## State

See [`UIActorMenu.h`](UIActorMenu.h.md).

## How a UI action becomes a game action

**Contract** — Six senders, one shape. Each builds a packed event addressed to an entity,
appends its payload, and submits it; then plays a sound and clears the highlight state.

```text
FUNCTION send_event_item_to_slot(item, recipient, slot)
  # If the item is not yet owned by the recipient, transfer ownership
  # FIRST, as a separate pair of events -- the slot event is addressed
  # to the item's parent, and that parent must already be the recipient.
  IF item.parent != recipient
    move_item_from_to(item.parent, recipient, item.id)

  event = new event(kind: player puts item in slot, to: item.parent)
  event.append(item.id)
  event.append(slot)
  submit(event)

  play the "to slot" sound
  clear highlights
```

The six:

| Sender | Event kind | Payload |
|---|---|---|
| to slot | player puts item in slot | item id, slot index |
| to belt | player puts item on belt | item id |
| to bag | player puts item in bag | item id |
| eat | player uses item | item id |
| drop | ownership rejected | item id |
| activate slot | player activates slot | slot index |

**Invariants**

- **Ownership transfer precedes placement.** Taking a gun from a corpse and equipping it is
  two ownership events followed by a slot event, in that order, and the slot event is
  addressed to the *new* parent. Reordering loses the item.
- **Drop asserts the actor already owns the item.** You cannot drop what is not yours; the
  gesture that looks like dropping somebody else's item is really two moves.
- In multiplayer a dropped item is additionally marked untradeable, so it cannot be
  re-acquired through the trade path in the same round.
- `move_item_from_to` — defined in
  [`UIActorMenuDeadBodySearch.cpp`](UIActorMenuDeadBodySearch.cpp.md) — is a *sell* event to
  the source followed by a *buy* event to the destination. Transfer between two inventories
  is expressed in the trade vocabulary even when no trade is happening, because that pair is
  the only ownership move the entity event system knows.

## `OnInventoryAction` — the reverse direction

**Contract** — The game tells the screen an item was acquired or lost. This is the only path
by which the screen learns about changes it did not cause, and it must leave the lists
matching the inventory exactly.

```text
FUNCTION on_inventory_action(item, kind)
  CASE acquired (bought, or taken):
    # Decide where the item BELONGS from its recorded place.
    place = item.current_place
    IF item's natural slot is the grenade slot
      place = bag                       # grenades live in the bag and are activated, not slotted

    target = the list for place: slot list / belt list / actor bag / loot bag
             (bag vs loot bag decided by whether the actor is the parent)

    already_there = false
    FOR EACH list IN every list that can hold an actor's item
      IF the item is in this list
        IF list != target THEN remove it from that list and destroy the cell
        ELSE already_there = true

    # A loot-mode item still showing in the loot list is being taken; the
    # loot list owns it and no duplicate is made.
    IF mode = loot AND the item is in the loot list THEN STOP

    IF NOT already_there
      target.add(new cell for item)
    refresh the quick slots

  CASE lost (sold, or dropped):
    # If the item being removed is the one under the cursor, abandon the drag.
    IF a drag is in flight AND it holds this item
      destroy the drag

    FOR EACH list IN every list that can hold an actor's item
      IF removing the item from this list succeeded THEN BREAK

    refresh the quick slots

  re-derive placement-dependent state           # prices, weights, highlights
```

**Invariants** — The acquire path searches *every* list and removes the item from all but the
target, because the same item can legitimately have been showing in a different list a moment
earlier (a trade side, a loot bag) and duplicates are the classic failure of an optimistic UI.

Abandoning a drag whose subject has been destroyed is not optional: the floating widget holds
a pointer to a cell whose item is gone.

## `InitInventoryContents` — filling the lists

**Contract** — Empty everything, then rebuild from the actor's inventory:

```text
FUNCTION init_inventory_contents(bag_list, only_bag)
  clear all lists; release the pointer capture; hide the context menu; clear the selection

  items = only_bag ? actor.inventory.available_items()   # the ruck as the game sees it
                   : actor.inventory.ruck                 # the raw ruck container
  sort items by descending footprint area                 # see below

  FOR EACH item IN items
    skip a multiplayer player's own bag object
    bag_list.add(new cell for item)
    IF mode = trade THEN tint the cell by whether the partner will take it

  IF only_bag THEN RETURN                 # the caller owns the slots

  fill each slot list from the inventory's slot occupants
  fill the belt list from the belt
  refresh the quick slots
```

**Invariants** — The **sort by descending footprint** is the load-bearing line. Cells are
packed into the grid greedily in insertion order (see
[`UIDragDropListEx.cpp`](UIDragDropListEx.cpp.md)); inserting the largest first is what keeps
the grid from fragmenting into unusable gaps. Remove the sort and a full inventory stops
fitting in the same grid it fitted in before.

Slots are filled explicitly, one call per slot, not by iteration — because *which* slots exist
is game-dependent, and because a slot marked **persistent** is deliberately skipped. A
persistent slot's item is permanently attached to the actor (the knife, the binoculars, the
detector, the personal terminal in some games) and must not appear as a movable cell.

Slots beyond the fixed set are swept by index, which is how a mod adds slots.

## `ToSlot` — the most intricate move

**Contract** — Put a cell's item into a numbered slot, optionally forcing the slot's current
occupant out to make room. Returns whether it happened.

```text
FUNCTION to_slot(cell, force, slot) -> bool
  item = cell.item
  ours = (item.parent = actor)

  # A sealed outfit forbids a helmet outright.
  IF slot = helmet AND actor's outfit has no helmet opening THEN RETURN false

  IF actor.inventory.can_put_in_slot(item, slot)
    target = the list for that slot
    IF there is no such list THEN RETURN true      # a slot with no widget still accepts

    # Putting on an outfit that seals the head must first remove the helmet.
    IF slot = outfit AND the outfit seals the head
      IF the helmet list holds exactly one item THEN move it to the bag

    IF ours THEN actor.inventory.place_in_slot(slot, item)

    moved = source.remove(cell)
    # A stack cannot go into a slot: the head goes, the rest stay behind.
    WHILE moved has children
      source.add(moved.pop_child())
    target.add(moved)

    emit item-to-slot event
    emit activate-slot event
    IF slot = outfit THEN move belt artefacts to the bag   # capacity may have shrunk
    RETURN true

  # The slot is occupied.
  IF NOT force THEN RETURN false
  IF the slot is persistent AND is not the detector slot THEN RETURN false

  # Try the alternative weapon slot first in the game that has two
  # interchangeable ones -- swapping is worse than using the empty one.
  IF an alternative slot exists AND accepts the item THEN RETURN to_slot(cell, force, alternative)

  occupant = actor.inventory.item_in_slot(slot)
  IF the slot's list is a separate widget
    it must hold exactly that occupant; move it to the bag, or give up
  ELSE                          # the slot IS the bag in this layout
    find the occupant's cell in the bag and move it to the bag (a no-op that re-seats it)

  result = to_slot(cell, force: false, slot)     # now it fits
  IF ours AND result AND slot = detector
    tell the detector whether the hands are currently free
  RETURN result
```

**Invariants**

- **The recursion always reduces force**: the retry after evicting passes `force = false`, so
  the eviction cannot loop.
- A stack is split on the way into a slot, head only. A slot holds one item by definition.
- `can_put_in_slot` is the inventory's rule, not the screen's. The screen asks and obeys; it
  never reimplements slot compatibility.
- The detector's "are your hands free" notification exists because a detector behaves
  differently when the other hand holds a weapon, and equipping it is the moment that changes.

## `ToBag`

**Contract** — Move a cell into the actor's bag, either at the cursor's position (when the
gesture was a drop) or at the next free grid position (when it was a double click or a menu
command).

Accepts when the inventory says the item fits **or** when it is already in the ruck and is
merely being moved between two different bag widgets — which is how an item crosses between
the inventory bag and the trade bag without a weight re-check.

Emits the to-bag event only when the item was not already in the ruck, or was not already
ours; re-seating an item we already hold is a pure UI move and must not generate an event.

## `ToBelt`

**Contract** — Move a cell onto the artefact belt, or refuse. When the belt is full **and the
gesture was a drop onto a particular cell**, the item under that cell is first moved to the
bag and the move retried — a swap. A double click onto a full belt simply fails, because there
is no cell to name.

**Invariants** — The swap path requires a cursor position, which is why `b_use_cursor_pos` is
threaded through every move: it is not a convenience, it distinguishes "put it somewhere" from
"put it *here*, displacing what is there".

## `ToQuickSlot`

**Contract** — Bind a consumable to one of the numbered quick slots. Refuses anything that is
not consumable, and refuses **anything larger than one grid cell**, because the quick slot row
is one cell tall and a larger icon would recurse through the reference list's placement.

The quick slot stores the item's **section name in a process-wide table**, not the item — the
key fires on whatever instance of that section the actor is carrying at the time. That is why
a used-up medkit's quick slot keeps working when another is picked up.

## `TryUseItem`

**Contract** — Use a consumable. Refuses anything that is not a medical item, an antidote, a
consumable or a bottle, and refuses an item the game says has no remaining use. When the item
belongs to somebody else — eating something straight out of a corpse — **the cell is removed
from its list first**, because the use event will destroy the item and the list must not
outlive it.

## `ActivatePropertiesBox` — the context menu

**Contract** — Build the right-click menu for the selected item and show it at the cursor,
clipped into the screen. The menu is assembled by a fixed sequence of contributors, each of
which may add entries; if none does, no menu appears.

In inventory and loot modes, in this order: **slot moves, weapon actions, attachment
attachment, use actions, playback, and (inventory only) drop.** In upgrade mode: repair only.
In trade mode: donate only, and only for items in the actor's own bag.

The ordering is the menu's reading order and is the decision; the contributors are:

### slot moves

Offers "move to slot" for an item with a real, non-persistent natural slot that is not already
occupied by this very item; "move to belt" when the belt would take it; and an unequip entry
whose **label depends on what is being unequipped** — outfit, helmet, or generic — and on
whether the shipped localization has the newer "unequip" string at all, falling back to the
older "move to bag" wording when it does not. Finally "wear" entries for an outfit, and for a
helmet only when no sealed outfit is worn.

### weapon actions

Offers detach entries for each attachment actually attached, and "unload magazine" when the
weapon **or any weapon in its stack** holds rounds — single player only, because multiplayer
handles magazines differently.

### attachment attachment

For a scope, silencer or launcher in hand, offers to attach it to whichever of the two weapon
slots will accept it, with the target weapon's display name appended to the label so two
offers are distinguishable.

### use actions

The label is data-driven with a hardcoded fallback chain:

```text
IF item.config has a "default use text" THEN use it
ELSE IF item is medical or an antidote   THEN "use"
ELSE IF item is a bottle                 THEN "drink"
ELSE IF item is consumable
  IF section is one of a small hardcoded set of drinks THEN "drink"
  ELSE IF section is one of a small hardcoded set of foods THEN "eat"
  ELSE "use"
```

**Notes** — The hardcoded section names are an embarrassment the source itself flags, and are
the reason the data-driven key exists. A rebuild should implement the configuration key and
may drop the name list; the shipped data relies on it for exactly those few items.

Then **four further custom use actions**, each named by a configuration key holding a script
function that returns the label — or nothing, to omit the entry. Four is a hard limit with no
reason beyond four distinct action tags being defined.

### playback, drop, repair, donate

Playback appears for a personal terminal that has a script attached. Drop appears for
non-quest items, plus a "drop all" variant when the item is stacked. Repair appears for an
outfit, weapon or helmet below 99% condition. Donate appears for non-quest items during trade.

**Invariants** — **Quest items can be neither dropped nor donated**, checked in both places.
That is a game rule, not a UI convenience.

The 99% repair threshold exists so that a nearly-pristine item does not offer a repair that
would cost money for no gain.

## `ProcessPropertiesBoxClicked`

**Contract** — Dispatch on the chosen entry's tag. The four custom use actions each first call
a *second* script function — the action's, distinct from the one that produced the label — and
use the item only if it returns true, which is how a script can refuse an action after
offering it.

Detaching an attachment detaches it from **every weapon in the stack**, not only the head,
because the stack merged weapons that all had it. Unloading a magazine likewise unloads the
whole stack.

Attaching an addon, in loot mode, additionally removes the addon from the loot list, because
the attach consumed it and the loot list learns nothing otherwise.

Repair returns immediately rather than falling through to the placement refresh, because it
opens a confirmation dialog and the refresh belongs after the answer.

## `AttachAddon` / `DetachAddon`

**Contract** — On a client, emit an attach or detach event and let the authority apply it; in
a single-process game, apply it directly. **On a client, detach returns without applying
locally** — the event is the whole of it. That asymmetry is the one place in the chapter where
the optimistic-UI rule is not followed, and it is deliberate: an attachment's removal spawns a
new item, which only the authority may do.

## `UpdateOutfit`

**Contract** — Re-derive the belt's capacity from the worn outfit, and the helmet slot's
availability from whether the outfit seals the head.

```text
FUNCTION update_outfit()
  belt.max_capacity = capacity for inventory.max_belt_width
  outfit = actor's outfit

  IF a helmet list exists
    helmet.capacity = outfit seals the head ? nothing : its authored maximum

  refresh the outfit protection readout

  IF the oldest game's rules
    belt.capacity = its authored default          # the belt is fixed there
    RETURN

  IF no outfit
    move every belt artefact to the bag
    belt.capacity = nothing
    RETURN

  belt.capacity = capacity for inventory.current_belt_width
```

**Invariants** — **Taking off an outfit empties the belt into the bag.** Artefacts are carried
by the outfit, not by the actor, and a rebuild that leaves them on a naked belt has changed a
game rule. The oldest game is exempt because its belt is not outfit-derived.

Capacity is set to zero rather than the list being hidden, so the slots are visibly there but
visibly unusable.

## `RefreshCurrentItemCell`

**Contract** — A script-facing operation: re-seat the selected cell so that its stack is
re-evaluated. It removes the stack head, re-adds every child as an independent cell, then
re-adds the head at the cursor — letting the list's auto-grouping re-merge whatever still
matches. This is how a script that changed an item's condition makes the inventory notice that
it no longer stacks with its neighbours.

## `DropAllCurrentItem` / `DropAllItemsFromRuck`

**Contract** — Drop a whole stack, or the whole bag. Both skip quest items; the bag sweep
takes a flag that overrides even that, and exists for development. Both pop children before
the head, so every dropped item is its own entity by the time its event goes out.

## `FindItemInList` / `RemoveItemFromList`

**Contract** — Locate a cell by the item it carries, searching stack children before stack
heads, and remove it. Searching children first matters: an item merged into a stack is found
by its own identity, not by the stack's.
