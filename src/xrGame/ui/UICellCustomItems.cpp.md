# src/xrGame/ui/UICellCustomItems.cpp

> What makes an inventory cell look like the item it holds: the icon taken from an atlas by
> grid coordinates, the layers and attachment icons composited over it, and the three
> different answers to "may these two stack".

**Needs** — [`UICellCustomItems.h`](UICellCustomItems.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md)
**Used by** — [`UICellCustomItems.h`](UICellCustomItems.h.md)
**Tier floor** — T2: builds widget trees from per-item configuration each frame

## Purpose

Three cell kinds, one file, because the differences between them are small and entirely about
presentation: where the icon comes from, what extra icons go over it, what the number in the
corner counts, and when two of them may merge. Splitting them would hide the fact that the
three stacking rules compose by narrowing.

## State

See [`UICellCustomItems.h`](UICellCustomItems.h.md). One invariant governs the whole file:

- **Every icon in the game lives in one atlas addressed by a 50×50 grid.** An item's
  configuration gives `inv_grid_x`, `inv_grid_y`, `inv_grid_width`, `inv_grid_height` in
  *cells*; multiplying by 50 gives the texture rectangle, and the width/height pair is
  simultaneously the item's **footprint in the inventory grid**. One number does both jobs,
  which is why an item's icon can never be a different shape from the space it takes up.

The 50 appears in this file as a literal and in the toolkit as a named constant; it is frozen
by the shipped icon atlas.

## `CUIInventoryCellItem` — construction

**Contract** — Take the item's grid rectangle from its configuration: the footprint becomes
the cell's grid size, and the same numbers scaled by 50 become the texture sub-rectangle into
the shared equipment atlas. The picture is drawn stretched, so the same icon works at any
canvas scale.

Then discover the item's **icon layers**:

```text
FUNCTION discover_layers(item)
  FOR i IN 0 .. 254
    key = i + "icon_layer"
    IF item.config has no line named key THEN BREAK      # dense, zero-based, stops at first gap
    section = item.config[key]                            # the icon to composite
    offset  = (item.config[key + "_x"], item.config[key + "_y"])
    scale   = item.config[key + "_scale"] or 1
    record layer(section, offset, scale)
```

**Invariants** — The key set is **dense and zero-based** and the scan stops at the first
missing index, so a configuration with layers 0 and 2 silently loses layer 2. The offset keys
are required once the layer key exists; the scale key is optional and defaults to 1. The upper
bound of 255 is a safety stop, not a meaningful limit.

**Notes** — A layer names another configuration section and reads *that* section's grid
coordinates, so a layer is "draw this other item's icon over mine". That is what gives modded
items composite icons — a backpack with a visible bedroll, a weapon with a painted finish —
with no engine change.

## `InitLayer` — placing one layer

**Contract** — Create (or reposition) the static that draws one layer, sized and offset
relative to the *cell's* current size rather than to the atlas, so layers track the cell when
the canvas scales or the cell rotates. Returns the static, creating it on first use.

```text
FUNCTION init_layer(static, section, offset, rotated, scale) -> static
  IF static does not exist
    create it, attach as a child, give it the equipment atlas and the cell's tint

  # Base scale converts atlas units into this cell's units. When the cell
  # is drawn rotated the two axes swap, which is why there are two arms.
  IF cell is rotated
    base = (cell.height / (50 * grid.x), cell.width  / (50 * grid.y)) * scale
  ELSE
    base = (cell.width  / (50 * grid.x), cell.height / (50 * grid.y)) * scale

  size     = (section.grid_width, section.grid_height) * 50
  tex_rect = section's grid rectangle * 50
  size     = size * base

  IF rotated
    static.size = (size.y, size.x)                  # swap
    # Re-express the authored offset in the rotated frame: the authored
    # x becomes a distance from the far edge, and the authored y becomes
    # the new x, corrected by the canvas aspect factor.
    offset = (offset.y * base.x,
              cell.height - offset.x * base.x - size.x)
    offset.x = offset.x * canvas_x_scale
  ELSE
    static.size = size
    offset = offset * base

  static.position = offset
  static.texture_rect = tex_rect
  static.stretched = true
  static.rotated = rotated; pivot about its own bottom-left
  RETURN static
```

**Invariants** — Both rotated arms use `base.x` for the offset, not `base.y`. That is not a
typo in the original: after a quarter turn the layer's *authored* x and y are both measured
along what is now the cell's height axis, which shares the x scale factor.

The aspect correction is applied to the rotated offset only, because the toolkit's canvas
stretch is horizontal-only (chapter 15) and a rotated offset crosses from one axis to the
other.

## `CUIInventoryCellItem::Update`

**Contract** — Per frame: refresh the condition bar and the count text, then choose a tint and
apply it to the cell and every layer.

```text
FUNCTION update()
  inherited update                  # orientation, focus notification, upgrade marker
  update condition bar
  update count text

  tint = current tint
  IF this cell is a helper AND has no stack     -> tint = dim grey
  ELSE IF this cell or any of its stack is a helper -> tint = full white
  apply tint to self and to every layer

  re-place every layer against the cell's current orientation
```

**Notes** — A **helper item** is an inventory item that exists only to back a displayed stack
— the shipped games create them when an item must appear in two places at once. A lone helper
is drawn dimmed to mark it as not-really-there; a helper merged into a real stack is drawn
normally because the stack as a whole *is* really there.

Layers are re-placed every frame rather than on change, because the cell's orientation can
change at any time (a list can be re-laid-out) and there is no change notification.

## `CUIInventoryCellItem::UpdateItemText`

**Contract** — The stack count, minus helpers. The displayed count is the stack size plus one,
less the number of helpers in it; nothing is shown for a count of one with no helpers.

**Notes** — The original's arithmetic here is defective — an operator precedence slip makes
the helper count collapse to 0 or 1 rather than counting — and a rebuild should simply count
the helpers. The intent is unambiguous: helpers are not merchandise and must not inflate a
displayed quantity.

## `CUIInventoryCellItem::CreateDragItem`

**Contract** — Refuse the drag entirely — return nothing — if this cell **or any cell in its
stack** is a helper. A helper has no independent existence, so it cannot be picked up.
Otherwise build the base drag widget and re-create every icon layer on it, unrotated, so the
thing under the cursor looks like the thing that was picked up.

## `CUIInventoryCellItem::EqualTo`

**Contract** — Narrows the base footprint test with three further conditions, all required:

```text
FUNCTION equal_to(other) -> bool
  RETURN base_test(other)                                  # same grid footprint
     AND this.item.section = other.item.section             # same kind
     AND this.item.condition ~= other.item.condition        # within 1% of each other
     AND this.item.upgrades = other.item.upgrades           # same upgrade set
```

**Invariants** — The condition tolerance is **1%**. Exact equality would leave the inventory
littered with near-identical singletons after any use; a wider tolerance would silently merge
items the player deliberately keeps apart, and the merged stack's condition is whichever cell
happens to be the head. One percent is the compromise, and it is visible in gameplay.

Upgrades must match exactly, with no tolerance — two upgraded weapons are different objects.

## `CUIAmmoCellItem`

**Contract** — Stacks with ammunition of the same section (a further narrowing on top of the
inventory rule, and redundant with it in practice). Refuses to produce a drag widget when it
is a helper.

Its one real behaviour is `CalculateAmmoCount`: the displayed number is the **sum of rounds
across the whole stack**, skipping helpers, not the number of boxes.

```text
FUNCTION calculate_ammo_count() -> int
  total = (this is a helper) ? 0 : this.item.rounds_in_box
  FOR EACH child IN stack
    IF child is not a helper THEN total = total + child.item.rounds_in_box
  RETURN total
```

**Invariants** — This is why ammunition is a distinct cell kind at all. The player thinks in
rounds; the inventory stores boxes; the cell is where the two are reconciled. A rebuild that
shows box counts on ammunition has changed the game's interface, not its rendering.

The count is suppressed entirely when an overdraw hook is installed — in the buy menu the
corner is occupied by a price instead.

## `CUIWeaponCellItem`

**Contract** — A weapon composites up to three attachment icons over its own. Each attach
point that the weapon's configuration declares carries an authored offset, read at
construction and **re-read** on every attach or detach, because a weapon's offsets can change
when its upgrades change.

```text
FUNCTION update()
  was_rotated = this cell is rotated
  inherited update
  force_reinit = (was_rotated != this cell is rotated)

  FOR EACH point IN {silencer, scope, grenade launcher}
    IF weapon does not support point THEN CONTINUE
    IF weapon has point attached
      IF no icon for point OR force_reinit
        create icon; re-read offsets; place it
    ELSE
      destroy the icon if there is one
```

**Invariants** — The rotation change is the reason `force_reinit` exists: an attachment icon
placed for an upright cell is wrong for a rotated one, and nothing else would notice the
change. The icons are created and destroyed as attachments come and go rather than being
shown and hidden, so a weapon carries no widgets for attachments it does not have.

`OnAfterChild` re-places all three against the *new* owning list's orientation, because moving
a weapon from the bag into a belt can rotate it.

`EqualTo` narrows the inventory rule with the attachment set: two weapons must have the same
attachments attached, and if both carry scopes the **scope must be the same scope** — because
the scope changes the weapon's behaviour and the player must not have two different sights
silently merged.

`CreateDragItem` re-creates each present attachment icon on the drag widget, unrotated.

`Draw` additionally draws the upgrade marker after the children. That is a layering fix: on a
weapon the attachment icons would otherwise cover the marker.

## `CBuyItemCustomDrawCell`

**Contract** — An overdraw hook that prints a short string at a cell's top-left corner in a
given font, converting from canvas coordinates to screen coordinates itself and flushing the
font immediately. At most 15 characters plus a terminator; the caller is asserted.

**Notes** — It flushes its own font rather than queueing into the frame's glyph batch, which
is why it can draw over cells the text layer has already flushed. That is the cost of drawing
outside the toolkit's batching discipline, and it is paid once per priced cell in the buy
menu.
