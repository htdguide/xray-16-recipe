# src/xrGame/ui/UICellItem.cpp

> One item sitting in an inventory grid: how it stacks, how a press on it becomes a drag, and
> what the floating thing under the cursor is while the drag lasts.

**Needs** — [`UICellItem.h`](UICellItem.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIHelper.h`](UIHelper.h.md) · [`../../xrUICore/Cursor/UICursor.h`](../../xrUICore/Cursor/UICursor.h.md) · [`../../xrUICore/ProgressBar/UIProgressBar.h`](../../xrUICore/ProgressBar/UIProgressBar.h.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`UICellItem.h`](UICellItem.h.md)
**Tier floor** — T2: widget tree, per-frame draw, no device-facing layout

## Purpose

This is the atom of every inventory-shaped screen in the game. A cell is simultaneously four
things: a picture of an item, the head of a stack of identical items, the source of a drag
gesture, and a carrier of highlight state that several different rules write into. Keeping
them in one widget is what lets the inventory, trade, loot and quick-slot screens share every
gesture.

The file also holds the drag widget, because the two are inseparable: a cell creates its own
drag representation, and the drag representation reports its drop back through the cell.

## State

See [`UICellItem.h`](UICellItem.h.md). Two invariants are worth stating here because they are
enforced by scattered assertions rather than by the type:

- **A stack is exactly one level deep.** A cell pushed as a child must itself have no
  children, and the head of a stack is never pushed onto another stack. Every merge and split
  asserts this.
- **`m_pData` identifies the item, the widget does not.** A split may swap payloads between
  the head and a child rather than moving widgets (see `PopChild`), so code that remembers a
  widget across a split is remembering the wrong item.

## `init`

**Contract** — Every cell, in every screen, builds its decorations from **one** shared layout
document, read at construction: a stack-count text, an upgrade marker, and a condition bar.
The document is optional — a data set without it yields a bare cell with no decorations rather
than a failure — and each element within it is optional too.

**Notes** — Reading one document per cell construction, rather than once per screen, is
wasteful and is how the original does it; the document is cached by the filesystem layer,
which is why it survives. A rebuild should hoist it.

The condition bar is looked up under **two** spellings, the correct one and a misspelling that
shipped in real data. That is not a decision, it is compatibility with a typo that mods
depend on; a rebuild targeting the shipped data owes both names.

The upgrade marker's authored position is remembered separately from its current position,
because the marker slides sideways when a stack count appears next to it — see `Update`.

## `Update`

**Contract** — Called once per frame while the cell is in a list. Does four things, in order,
and the order is the decision:

```text
FUNCTION update()
  # 1. Orientation follows the owning list. A list may lay items out
  #    rotated (a vertical belt); the cell rotates with it, pivoting
  #    about its own bottom-left so the footprint stays in the grid.
  IF owner_list.vertical
    enable heading; heading = quarter turn
    pivot = (0, 0) about (0, height)
  ELSE
    reset pivot

  inherited update                # children, animation, hover timestamps

  # 2. Tell the screen the cursor is dwelling on this cell -- but only
  #    if the cursor is inside the LIST's client area, not merely inside
  #    the cell. A cell scrolled half out of view must not claim focus.
  IF cursor over this cell AND cursor inside owner_list.client_area
    notify message target: item focused update

  # 3. The upgrade marker is shown only for an item that has upgrades,
  #    and slides right by the stack-count text's width when the cell
  #    also shows a count, so the two never overlap.
  has_upgrade = payload has any upgrades
  IF stack is non-empty
    marker.position = authored position shifted right by count text width + 2
  ELSE
    marker.position = authored position
  marker.shown = has_upgrade
```

**Invariants** — Step 2's *list* client-area test, not the cell's own rectangle, is what makes
tooltips behave at the edge of a scrolled list. The toolkit's hover has no regard for overlap
(chapter 15); this is the game layer adding the clipping the toolkit deliberately omits.

## `OnMouseAction` — how a press becomes a drag

**Contract** — Translates pointer events into the four notifications a drag-and-drop list
understands, and decides whether the event is consumed.

```text
FUNCTION on_mouse(action) -> consumed
  IF action = left button down
    notify: item left-clicked
    notify: item selected
    press_anchor = this            # process-wide
    RETURN false                   # NOT consumed: selection must also reach the list

  IF action = mouse move
    # A drag begins only if the button is still physically down AND the
    # press began on THIS cell. Without the anchor, dragging the cursor
    # across a grid with the button held would start a drag on every
    # cell it crossed.
    IF left button is physically held AND press_anchor = this
      notify: item drag
      RETURN true

  IF action = left button double click
    notify: item double-clicked
    RETURN true

  IF action = right button down
    notify: item right-clicked
    RETURN true

  press_anchor = none              # any other event abandons the gesture
  RETURN false
```

**Invariants** — The anchor is a **single process-wide value**, not per-list and not per-cell.
That is correct rather than sloppy: there is only ever one pointer, so there is only ever one
press in flight. A rebuild with multi-pointer input needs one anchor per pointer.

The button-held test reads the *physical* button state rather than trusting a press/release
pairing. That is deliberate: the release may have been delivered to a different widget, or to
nothing at all if the window lost focus mid-drag, and a stuck drag is far worse than a missed
one.

Left-button-down returning *not consumed* is the subtle half: the list below also needs the
event, to move its selection.

## `OnKeyboardAction`

**Contract** — Two paths. A cell may carry an **accelerator** — a key that activates it
wherever the pointer is; that is how the quick slots work, and pressing it is equivalent to a
double click. Otherwise, if the cursor is over the cell, the two gamepad/keyboard UI actions
"accept" and "action 1" are translated into a double click and a right click respectively, at
the cursor's position, so that every cell gesture is reachable without a mouse.

## `CreateDragItem`

**Contract** — Build the floating widget for a drag that is starting on this cell. It takes
the cell's current absolute rectangle, its material and its texture sub-rectangle, so the
floating thing looks exactly like what was picked up.

The one piece of arithmetic is for a **rotated** cell:

```text
FUNCTION create_drag_item() -> DragWidget
  r = this.absolute_rect
  IF this cell is drawn rotated
    # Un-rotate the footprint and re-centre it on the cursor: the drag
    # widget is always drawn upright, so a belt item picked up sideways
    # must become a correctly proportioned upright icon under the cursor.
    w = r.height * canvas_x_scale          # aspect factor, see chapter 15
    h = r.width
    r = rect centred on cursor with size (w, h)

  RETURN DragWidget(this) initialised with material, r, texture rect
```

**Notes** — The `canvas_x_scale` factor is the toolkit's non-uniform aspect correction: a
quarter turn swaps the axes, and without the correction a rotated icon un-rotates to the wrong
proportions on a wide display.

## `UpdateConditionProgressBar`

**Contract** — Show or hide the small wear bar under the icon, and set its position. Hidden
unless the owning list wants condition bars *and* the item uses condition at all.

```text
FUNCTION update_condition_bar()
  IF no bar OR NOT owner_list.wants_condition_bars OR NOT item.uses_condition
    hide bar; RETURN

  fraction = item.condition                 # 0..1

  # An item measured in USES rather than wear -- food, medicine -- shows
  # remaining uses instead, and the mapping depends on how many uses it has.
  IF item is consumable with max_uses > 0
    IF max_uses < 8  THEN hide the bar's background
    IF remaining = 0 THEN fraction = 0
    ELSE IF max_uses > 8
      fraction = remaining / max_uses       # a continuous bar
    ELSE
      fraction = remaining * 0.125 - 0.0625 # one eighth per use, centred in its eighth
    turn off the bar's gradient             # discrete uses are not a health colour ramp

  place the bar at the bottom-left of the item's footprint, one unit in
  fraction = ceil(fraction * 13) / 13       # quantise to the bar's 13 drawn segments
  show bar
```

**Invariants** — Three magic numbers, all frozen by the shipped art:

- **13** is the number of segments the bar texture is drawn with. Quantising to it is what
  stops a bar from rendering a partial segment. Change the art, change the number.
- **8** is the cut between "few enough uses to show as discrete eighths" and "enough to show
  as a continuous fraction". Eight is the number of segments the *background-less* variant of
  the bar has, which is why the background is also suppressed below it.
- **0.125 / 0.0625** is one eighth per use, offset by half an eighth so that the filled region
  ends in the middle of the segment representing the last remaining use rather than at its
  edge.

## Stacking: `EqualTo`, `PushChild`, `PopChild`, `HasChild`, `UpdateItemText`

**Contract** — `EqualTo` is the base mergeability test: two cells may stack only if their grid
footprints match. Subclasses narrow it further (same section, similar condition, same
upgrades, same attachments — see [`UICellCustomItems.cpp`](UICellCustomItems.cpp.md)), and
every narrowing is an `AND`.

`PushChild` merges a cell into this one's stack and refreshes the count text. `PopChild`
removes one and is where the subtlety lives:

```text
FUNCTION pop_child(wanted) -> Cell
  taken = last child            # always the LAST, regardless of which was asked for
  remove it from the stack

  IF wanted was named
    # The caller wants a specific ITEM, not a specific widget. Rather than
    # search the stack, swap the payloads: the widget that leaves carries
    # the wanted item, and the wanted widget stays holding the item that
    # would have left.
    IF taken is not wanted THEN swap taken.payload WITH wanted.payload
  ELSE
    # No specific item wanted: the HEAD's payload leaves, and the head
    # adopts the payload of the widget being removed. The head widget --
    # with its position in the grid, its selection state, its decorations
    # -- stays put.
    swap taken.payload WITH this.payload

  refresh count text
  taken.owner_list = none
  RETURN taken
```

**Invariants** — Widgets are recycled; payloads move. The stack head keeps its identity, its
grid position and its highlight state across any number of splits, which is what makes
"drag one off the stack" feel stable. Code holding a widget pointer across a split is holding
a widget whose item has changed — that is the trap this design sets, and the reason
`m_pData` and not the widget is the item's identity.

The removed cell is asserted to have no children of its own, which re-states the one-level
invariant.

`UpdateItemText` writes the count as `x<n>` where *n* is the stack size **plus one** (the head
counts), and shows nothing at all for a stack of one. Subclasses override it: ammunition shows
a round count, not a box count.

## `CUIDragItem`

**Contract** — The floating representation of a drag. Constructed from the cell being dragged;
registers itself with the frame loop's render and update lists at a priority below every other
UI consumer, so it draws last and therefore on top of every screen. Unregisters on destruction
— and must, because the frame loop holds the only other reference.

`Init(material, rect, texture rect)` sets it up as a stretched picture at 170 of 255 alpha, so
the dragged thing reads as in-flight rather than placed. It also records the **offset from the
cursor to the picked-up rectangle's top-left**, and that offset is what the drag preserves:

```text
FUNCTION draw()
  # Re-anchor to the cursor every frame rather than on mouse-move events,
  # so the widget cannot lag or stick when events are coalesced or dropped.
  move so that top_left = cursor + pickup_offset
  inherited draw
  IF custom draw hook THEN hook.on_draw(this)
```

**Invariants** — Grabbing an icon by its corner and grabbing it by its middle must feel
different, and the offset is the whole of that. Re-deriving position from the cursor each
frame, rather than accumulating deltas, is what keeps it exact.

`OnMouseAction` handles exactly one event — left button up — and turns it into an "item
dropped" notification sent to the *originating cell's* message target. Every other event is
ignored. The drop target itself is not decided here: see `SetBackList`.

`SetBackList(list)` records which list the cursor is currently over and tells the old and new
lists that the drag left and entered. That notification is what lets a list highlight itself
as a valid target, and what lets the trash slot install its badge hook.

`GetPosition` returns cursor plus offset, for a caller that needs the drop point before the
widget has drawn.
