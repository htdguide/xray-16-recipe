# src/xrGame/ui/UIOutfitSlot.cpp

> The armour slot: it accepts an item like any other cell, and then hides the cell entirely
> behind a full-length picture of the actor wearing what is in it.

**Needs** — [`UIOutfitSlot.h`](UIOutfitSlot.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`CustomOutfit.h`](../CustomOutfit.h.md) · [`Actor.h`](../Actor.h.md) · [`Level.h`](../Level.h.md)
**Used by** — [`UIOutfitSlot.h`](UIOutfitSlot.h.md)
**Tier floor** — T3.

## Purpose

The inventory screen shows what the player is wearing as a person, not as a small grid icon.
This file is that substitution, and it is done by *overriding drawing*, not by replacing the
cell: the slot is a perfectly ordinary one-cell drag target underneath, so every placement
rule, drag rule and validity check in the inventory screen applies unchanged.

## State

```text
RECORD OutfitSlot EXTENDS DragDropList
  background     : Static   # the portrait; the only thing ever drawn
  default_outfit : text     # the portrait used when nothing is worn and there is no actor
```

**Invariants** — the portrait is resized to the whole slot and stretched on every refresh, so
the slot's authored size, not the texture's, decides the picture's shape. The cell contents are
never drawn: `Draw` draws the portrait and returns.

## `SetOutfit` — choosing the portrait

**Contract** — refresh the portrait for whatever is now in the slot.

```text
FUNCTION set_outfit(item)
  background.rect    <- the whole slot
  background.stretch <- true
  IF single player AND the slot is empty THEN
    actor <- the current view entity as an actor
    name  <- actor EXISTS ? actor's visual model name : default_outfit
    strip any leading directory path from name
    strip a trailing four-character extension from name
    background.bind(name)
  ELSE IF item EXISTS THEN
    background.bind(the worn armour's own full-size portrait name)
  ELSE
    background.bind("npc_icon_without_outfit")
  background.show_texture()
```

**Notes**

- **The empty single-player portrait is derived from the actor's model file name.** The path and
  the extension are stripped and what remains is used as a texture name, which works because the
  shipped portrait atlas names its entries after the models. That is an undocumented coupling
  between two asset sets and it is the file's one genuinely fragile decision: a rebuild that
  renames either set silently gets a missing texture.
- In multiplayer the actor-derived portrait is skipped entirely and the empty slot shows the
  fixed no-armour picture, because there the player's appearance is decided by team and class
  rather than by a persistent model.
- The armour's own portrait comes from the armour item, not from this file, so one slot serves
  every suit in the game.

## The four overrides

**Contract** — each of the three placement forms places the item through the base list, but
**only when there is an item**, and then refreshes the portrait; removal removes through the
base and refreshes the portrait as empty. Removal to the root is not supported.

**Notes** — the null guard exists because the inventory screen calls the placement with nothing
to mean "the slot is now empty", which the base list would not accept. So the override is
simultaneously a placement and a clear.
