# src/xrGame/ui/UIDragDropReferenceList.cpp

> The quick-use bar: a cell board whose cells hold not items but the *names* of items, persisted outside the screen, so a slot keeps pointing at "medkit" after the medkit is used up.

**Needs** — [`UIDragDropReferenceList.h`](UIDragDropReferenceList.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`UICellItemFactory.h`](UICellItemFactory.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/Cursor/UICursor.h`](../../xrUICore/Cursor/UICursor.h.md) · [`Inventory.h`](../Inventory.h.md) · [`InventoryOwner.h`](../InventoryOwner.h.md) · [`Actor.h`](../Actor.h.md) · [`actor_defs.h`](../actor_defs.h.md)
**Used by** — [`UIDragDropReferenceList.h`](UIDragDropReferenceList.h.md)
**Tier floor** — T3.

## Purpose

The player assigns four items to number keys and uses them without opening anything. That
assignment must survive the item running out, being dropped, or the screen closing — so what
a quick-use slot holds is a **configuration section name**, not an item.

This widget is the editor for that assignment. It reuses the cell board for its geometry and
its drag interaction, and overrides placement and removal so that every change writes through
to the persistent slot table rather than to the board.

## State

```text
RECORD ReferenceList EXTENDS DragDropList
  references    : list<Picture>   # one per cell, in the board's row-major order
  labels        : list<Label>     # the key hint drawn beside each cell
  label_id_form : text            # localization identifier pattern for those hints

# the authority is elsewhere:
#   quick_use_slots : list<text>  # a small fixed array of section names, owned by the
#                                 # actor's definitions and saved with the player
```

**Invariants**

- `references[capacity.x * row + column]` is the picture for that cell — the same row-major
  index the board uses, and the same index into the persistent slot table. **Three
  structures share one index**; a rebuild that reorders any of them breaks the other two.
- A reference picture's alpha encodes the slot's state, and this is the whole display logic:
  fully opaque means the named item is in the inventory right now, partly transparent means
  the slot is assigned but the item is absent, fully transparent means the slot is empty.

## Construction

**Contract** — build one picture per cell at that cell's position and size, named
`cell_item_reference`, and bind a double-click handler to that name. Optionally also build
one key-hint label per cell from a layout, rebased from absolute into the list's coordinates
for the same clipping reason the base list rebases its decorations.

**Notes** — label construction indexes the layout element by `column + row + 1`, which is
correct only for a single row or a single column. The shipped quick-use bar is one row of
four, so it works; a two-dimensional reference list would build two labels for the same
element and skip another. A rebuild indexes by the row-major cell index.

## Refilling from the slot table

**Contract** — the one operation that matters. For each cell, read the assigned section
name and render it:

```text
FUNCTION reload_references(owner)
  cancel any drag in flight
  clear the board, destroying its cell items
  FOR EACH cell
    name := quick_use_slots[cell_index]
    IF name is empty
      reference[cell_index].alpha := 0             # slot unassigned
    ELSE IF owner.inventory holds an item of that section
      place a real cell item for it at this cell   # opaque, draggable, usable
    ELSE
      load the icon straight from the section's configuration
      reference[cell_index].alpha := 100 of 255    # ghosted: assigned but not carried
```

**Notes** — the ghost is the point of the whole class. A slot that cannot be satisfied keeps
its picture so the player sees *what the key is for*, and the icon in that case is read from
the game's configuration rather than from any item — the section names its sub-rectangle of
the shared equipment icon atlas in grid units, multiplied by the atlas's cell size. That is
the same atlas addressing every item icon in the game uses; see
[`UIInventoryUtilities.cpp`](UIInventoryUtilities.cpp.md).

The ghost alpha is 100 of 255 — about forty percent. No finer reason is recoverable; it is a
look.

## Placement overrides

**Contract** — placing at a cell copies the incoming item's material and icon rectangle onto
that cell's reference picture and makes it opaque, then places the item on the board only if
the cell does not already hold it. Placing at a point differs from the base list in one
decisive way: **an occupied cell is not refused, it is replaced.** The occupant is removed
and the new item takes the cell.

**Notes** — replace-on-drop is right here and wrong in an inventory. A quick-use slot is a
*binding*, and dropping a new item on a bound key means rebinding it; there is nowhere else
for the old one to go, because it was never really there.

Stacking is bypassed entirely — the base list's stacking would merge two slots holding the
same section into one, which is exactly what a player assigning the same item to two keys
does not want.

## Removal overrides

**Contract** — removing an item also **clears the persistent slot** and blanks its reference
picture. The override returns nothing, deliberately: the base list returns the item so the
caller can re-place it, and there is nothing to re-place here — the item goes back to being
an ordinary inventory item.

## Reordering within the bar

**Contract** — dropping an item that came from *another* list falls through to the ordinary
cross-list transfer. Dropping an item that came from this list **swaps the two slot names**
in the persistent table and rebuilds the whole bar from it.

```text
FUNCTION on_drop(item)
  IF the screen's drop hook handles it THEN cancel the drag; RETURN
  IF item came from a different list THEN delegate to the base list; RETURN
  from := cell the item occupies
  to   := cell under the cursor
  swap quick_use_slots[from], quick_use_slots[to]
  reload_references(actor)
```

**Notes** — reordering goes through the slot table and a full rebuild rather than through the
board, because the table is the authority and the board is a view of it. That is the
reordering rule a rebuild must copy: **every mutation writes the table and re-reads it**, so
the two can never drift.

A drop onto empty space inside the bar swaps a slot with an empty one, which moves the
binding — the swap needs no special case for that.

## Double click

**Contract** — double-clicking a *reference picture* (not an item) unassigns that slot:
removes any live cell item, clears the slot name, and blanks the picture.

**Notes** — the handler is bound to the picture's name, so it only fires on cells whose item
is absent or on the empty background of a cell; a live item's own double click goes to the
item handler instead. Two different unbind gestures over one board, separated by which widget
is on top.

## Key hints

**Contract** — each label shows the key bound to its slot, looked up in the localization
table by a per-slot identifier, and then **truncated to at most three characters, or two if
the third is a comma**.

**Notes** — the truncation is display fitting, not parsing: the localized binding string can
be a list ("1, NUM1") and the label is a few units wide. Cutting at a comma keeps the first
alternative whole instead of showing a dangling separator. A rebuild that lays out text
properly should split on the separator and take the first entry rather than count characters.
