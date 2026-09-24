# src/xrGame/ui/UIDragDropListEx.cpp

> The cell grid every inventory-like screen is made of: a rectangular board of cells, items that occupy rectangles of cells, stacking of identical items, one process-wide drag in flight, and a hand-built batched grid background.

**Needs** — [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UICellItem.h`](UICellItem.h.md) · [`xrUICore/ScrollBar/UIScrollBar.h`](../../xrUICore/ScrollBar/UIScrollBar.h.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/ProgressBar/UIProgressBar.h`](../../xrUICore/ProgressBar/UIProgressBar.h.md) · [`xrUICore/Cursor/UICursor.h`](../../xrUICore/Cursor/UICursor.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md) · [`Inventory.h`](../Inventory.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`UIDragDropListEx.h`](UIDragDropListEx.h.md)
**Tier floor** — T2: it emits its grid background as a hand-filled vertex batch with explicit texture coordinates rather than through the widget quad emitter, and the per-frame cost is proportional to visible cells.

## Purpose

Every screen in this chapter that shows things the player can pick up — the inventory, the
belt, trade, the upgrade bench, a corpse, a container — is built from this one widget. It
owns a **grid of cells** and a set of **items that occupy rectangles of cells**, and it
answers the four questions those screens need: where does a new item go, may this item go
*here*, what is under the cursor, and what happens when something is dropped on me.

It is deliberately ignorant of what an item *is*. Items carry an opaque payload, and every
policy decision — may this be dragged, may this be dropped, what does a double click mean —
is delegated to a hook the owning screen installs. That is the mechanism by which a UI
action becomes a game action: the list performs the *placement*, the screen performs the
*transaction*.

The file also contains the cell container, which is the list's inner scrolling surface. The
split is real: the list is the viewport, the scroll bar and the policy; the container is the
board, its geometry and its drawing.

## State

```text
RECORD Cell
  item     : optional<Item>   # the item occupying this cell
  is_main  : bool             # true only in the item's top-left cell
  # two cells are considered equal when they hold the same item — this is what
  # collapses a multi-cell item to one entry when gathering a draw range

RECORD CellBoard                     # the inner container
  capacity      : (int, int)         # columns, rows
  cell_size     : (int, int)         # canvas units per cell
  cell_spacing  : (int, int)         # canvas units between cells
  cells         : list<Cell>         # row-major: index == capacity.x * row + column
  grid_material : Material           # one atlas, four horizontal frames

RECORD DragDropList EXTENDS Window   # the viewport
  board             : CellBoard
  scroll            : ScrollBar
  selected          : optional<Item>
  start_capacity    : (int, int)     # the authored capacity, restored on reset
  max_capacity      : (int, int)     # how far growth is allowed; drives the blocker
  virtual_alignment : (int, int)     # 0 near, 1 centre, 2 far, per axis
  highlighter       : optional<Picture>   # tiled once per visible cell
  blocker           : optional<Picture>   # tiled once per locked cell
  condition_bar     : optional<ProgressBar>
  back_colour       : colour
  flags             : { group_similar, auto_grow, custom_placement,
                        vertical_placement, always_show_scroll, virtual_cells }
  hooks             : ten per-event veto callbacks (see below)
```

**Invariants**

- Every cell covered by an item points at that item; **exactly one** of them — the
  top-left — is marked main. Removal clears the whole rectangle, so a partially cleared item
  is a corruption the board cannot detect.
- An item's on-screen size is its grid size times the cell size; it is never scaled to fit.
- The cell vector is resized on every capacity change, which **discards placement**. Nothing
  re-places items after a resize, so capacity is only changed while the board is empty or
  immediately before a full refill.
- **At most one drag is in flight in the whole process.** The drag item is a single shared
  slot, not a per-list one — see below.
- The board's own size is `(cell_size + spacing) * capacity - spacing`, i.e. spacing between
  cells but not outside them. Every capacity, size or spacing change recomputes it and then
  re-derives the scroll range.

## The single drag in flight

**Contract** — dragging starts by asking the item to produce a *drag proxy* — a floating
widget that follows the cursor — and giving the screen's root pointer capture to it. The
proxy is held in a slot shared by every list in the process. A drag therefore cannot be
started while another is running, and any list can inspect the drag in progress.

**Notes** — the shared slot is not laziness. A drag crosses lists — that is its entire
purpose — and the receiving list must be able to see a drag it did not start. A rebuild
gives the *screen* one drag slot instead of making it global; global is the original's
shortcut and is visible as one restriction, namely that two screens cannot both have a drag
running, which no shipped screen wants anyway.

Each frame, every list tests the cursor against its own absolute rectangle and claims or
releases the drag proxy's **back list** — the list the item would return to if released now.
Claim is first-come and release is self-only, so overlapping lists produce whichever claimed
first; the shipped layouts do not overlap them.

Pointer capture is released and the proxy destroyed only by the list that owns the
*selected* item, which is how a cancelled drag finds its way home.

## The ten hooks

**Contract** — ten hook points, each taking the item and returning whether the screen has
handled it. **A hook that returns "handled" suppresses the list's own default behaviour
entirely.** The events are: drag started, dropped, selected, left click, right click, double
click, focus received, focus lost, focused-update (once per frame while focused), and a
drag-crossing notification that carries the proxy and whether this list is gaining or losing
it.

**Notes** — this is the veto pattern, and it is the whole reason a purely geometric widget
can enforce game rules it knows nothing about. The trade screen refuses a drop the player
cannot afford by returning "handled" from the drop hook and doing nothing; the inventory
moves an item into a slot by returning "handled" and performing the move itself; a list with
no hooks installed behaves as a plain container. A rebuild that replaces the hooks with
subclassing must keep the *veto* semantics, not merely the notification.

Left and right click deliberately do **not** change the selection — the commented-out
selection call in each is load-bearing by its absence, because a context menu opened with
the right button must not move the highlight.

## Placement

**Contract** — three ways to place an item, all of which first try to *stack* it:

```text
FUNCTION place_auto(item)
  IF stack_into_similar(item) THEN RETURN
  place_at_cell(item, first_free_cell(item.grid_size))

FUNCTION place_at_point(item, absolute_point)   # used on drop: "where the cursor was"
  IF stack_into_similar(item) THEN RETURN
  cell := pick_cell(absolute_point)
  IF cell is valid AND room_is_free(cell, item.grid_size)
    THEN place_at_cell(item, cell)
    ELSE place_auto(item)

FUNCTION place_at_cell(item, cell)
  REQUIRE room_is_free(cell, item.grid_size)
  size := item.grid_size
  IF vertical_placement THEN swap size.x, size.y
  FOR EACH (x, y) IN size
    board.cell_at(cell + (x, y)).item    := item
    board.cell_at(cell + (x, y)).is_main := (x = 0 AND y = 0)
  item.screen_size := size * cell_size
  item.screen_pos  := cell * (cell_size + spacing)   # unless virtual cells; see below
  adopt(item); name it "cell_item"; bind it to this list
```

**Invariants** — an item is registered under the fixed child name `cell_item`, and every
hook in the list is bound to that name rather than to a widget. New items therefore need no
individual binding: **the name is the binding**. That name is frozen across this chapter.

**Notes** — the "vertical placement" flag swaps the *item's* footprint, not the board's.
A two-by-one rifle lies across two columns in the inventory and down two rows on the belt,
from one authored grid size. The swap is applied at every point that reasons about footprint
— placement, removal, free-space search — and forgetting it in one of them leaves ghost
cells.

## Finding a free cell

**Contract** — scan row by row from the top, left to right within a row, and take the first
position whose whole rectangle is free. Failing that: if the list may grow, add one row and
retry; otherwise compact and retry once; if it still fails, that is fatal.

```text
FUNCTION first_free_cell(size) -> cell
  IF vertical_placement THEN swap size.x, size.y
  FOR row IN 0 .. capacity.y - size.y
    FOR col IN 0 .. capacity.x - size.x
      IF room_is_free((col,row), size) THEN RETURN (col,row)
  IF auto_grow THEN grow_one_row(); RETURN first_free_cell(size)
  compact(); retry the same scan
  FAIL WITH "no room to place item"
```

**Notes** — row-major-from-the-top is what makes a player's inventory settle into the
familiar reading order, and it is the reason a freed cell in the middle is refilled before
the bottom row. Growth adds **rows only, never columns**: a cell board's width is authored
and its height is elastic, because the viewport scrolls vertically.

There is no shrink. The board never gives rows back on its own; capacity returns to the
authored value only through an explicit reset, which requires the board to be empty first.
An auto-growing list therefore keeps its high-water mark for as long as it holds anything.

**Compaction** empties the board — without destroying items — and re-places every item in
the order it happened to sit in the child list. It is first-fit repacking, not an optimal
one, and two large items can still fail to fit in a board with enough total free cells.

## Stacking

**Contract** — when grouping is on, a new item that is *equal to* an item already on the
board is absorbed into it as a child rather than given cells of its own. The stack's count
is its children plus itself. An item that already has children is never absorbed, so stacks
never nest.

Two exclusions, both about the game rather than the grid:

```text
FUNCTION may_stack(item) -> bool
  IF item is currently equipped in its own slot AND equipped-highlight is on
    THEN RETURN false            # an equipped round must stay visibly distinct
  IF item's configuration section declares "dont_stack"
    THEN RETURN false
  RETURN true
```

**Notes** — "equal to" is the item's own comparison, not identity: two ammunition boxes of
the same type and condition are equal. Excluding equipped items is gated on the same console
variable that draws the equipped highlight, so switching the highlight off also merges the
equipped item back into its stack — one setting with two effects, which is surprising and is
the original's behaviour.

The `dont_stack` flag is read from the item's configuration section on every comparison. A
rebuild caches it on the item; the lookup is per-candidate-per-insert here.

## Removal

**Contract** — removal has three cases, tried in order, and the order is the interesting
part:

```text
FUNCTION remove(item, force_root) -> Item
  # 1. the item is a member of somebody else's stack: pop it out of that stack
  FOR EACH placed IN board.items
    IF placed.has_child(item) THEN RETURN placed.pop_child(item)
  # 2. the item is a stack head and the caller wants one unit: pop one off the top,
  #    leaving the head where it is
  IF NOT force_root AND item.child_count > 0 THEN RETURN item.pop_child(any)
  # 3. the item itself leaves: clear its whole cell rectangle and detach
  clear_cells_of(item); detach(item); RETURN item
```

**Invariants** — whatever is returned has no children. The caller therefore always receives
a single unit unless it asked for the root.

**Notes** — case 2 is what makes dragging one round off a stack work without any explicit
"split" operation: dragging a stack head with `force_root` false yields one unit and leaves
the stack in place. `force_root` true is passed when the source and destination are the same
list, because moving a stack within one board should move the whole stack.

The cross-list drop transfers a stack by **moving the children out one at a time first and
the head last**, each one placed at the drag proxy's position. That means the destination's
own stacking rule re-forms the stack on arrival — possibly differently, if the destination
has grouping off or a `dont_stack` rule applies there.

## Hit testing

**Contract** — a point in absolute canvas units maps to a cell by subtracting the board's
origin and dividing by a per-axis stride. An out-of-range result is reported as the
invalid cell.

**Notes** — the stride used is `cell_size + spacing * (capacity - 1) / capacity`, not
`cell_size + spacing`. That is the *average* pitch over the board — total span divided by
cell count — rather than the pitch between adjacent cells, because the board has spacing
between cells but none after the last one. It is exact at the two ends and off by a fraction
of the spacing in the middle. With the shipped layouts spacing is zero or one unit and the
error never crosses a cell boundary; it would with a larger gap. Flagging it because a
rebuild that "corrects" this to the adjacent-cell pitch will not match the original's hit
testing at the board edges.

Drawing, in contrast, uses the adjacent-cell pitch — and applies the spacing **twice**, once
through the cell stride and once as an explicit spacing term. With zero spacing the two agree,
which is why the disagreement is invisible in the shipped data.

## Drawing the board

**Contract** — draw only what is visible, as one primitive batch, then the items on top,
all clipped to the client area.

```text
FUNCTION draw_board()
  client := list rectangle, minus the scroll bar's width when it is shown
  visible := cell range from the first row scrolled into view
             through the last row that intersects client
  IF highlighter is shown THEN draw it once per visible cell, offset by its spacing
  OPEN one triangle-list batch sized for the visible range
  FOR EACH visible cell
    frame := 0 normal | 1 selected | 2 marked-or-equipped | 3 armament-selected
    push two triangles at the cell's screen rectangle, textured with that frame
  CLOSE with the grid material
  PUSH scissor = client
  FOR EACH distinct item in the visible range
    IF it has not already drawn this frame THEN draw it
  POP scissor
  draw the blocker
```

**Invariants** — the grid atlas is four frames laid out horizontally, each a quarter of the
page wide and the full page tall. The selection state chooses the frame and **nothing else
does**: the cell's column and row are passed to the frame chooser and ignored, a vestige of a
scheme where the grid texture varied by position.

**Notes** — three details that are each load-bearing.

*One batch for the whole grid.* Chapter 15 states that the widget quad emitter deliberately
opens and flushes a batch per quad to preserve draw order. The grid escapes that rule by
going around the emitter: it is a uniform background behind everything, so there is no order
to preserve, and a board can be hundreds of cells. The half-unit offset on every vertex is
the same texel-centre correction the emitter applies.

*Items are deduplicated by identity when the draw range is gathered.* A multi-cell item
appears in several cells; the gather collapses adjacent duplicates. A per-frame stamp on the
item catches the rest, including the case where one item is reachable from two lists.

*The highlighter and blocker are ordinary picture widgets drawn many times.* Each is
positioned once, then drawn repeatedly at a per-cell offset, then restored. They are marked
"custom draw" so the normal tree traversal skips them — otherwise they would also draw once
at their own position. When the list is in virtual-cell mode they are drawn exactly once,
because there is no grid to tile across.

## The blocker: locked capacity

**Contract** — when a maximum capacity is configured and the current capacity is below it,
the blocker picture is tiled over the cells between the two, marking them visibly unusable.
When current and maximum agree on an axis, the range on that axis is taken as the last cell
only, so a fully unlocked board still shows nothing.

**Notes** — this is how the belt shows locked slots that an artefact container will later
unlock: capacity is the number of slots the player has *earned*, maximum is what the layout
drew. The pair is computed by `CalculateCapacity`, which turns a desired slot count into a
column/row shape consistent with the authored markup — square markup splits the count in
half on both axes, a wide markup keeps the authored height, a tall one keeps the authored
width, and a single row or column takes the count directly. A count that does not divide
evenly into the markup is fatal, which makes a bad belt-capacity setting a hard failure at
open rather than a silently truncated belt.

**Notes** — raising the maximum never shrinks anything, but lowering it below the current
capacity tries to clamp the current capacity to it and does so with a range whose lower bound
exceeds its upper bound. The result is not well defined. No shipped configuration lowers the
maximum, which is why this has never been seen; a rebuild should clamp to the maximum
plainly.

## Virtual cells

**Contract** — in virtual-cell mode an item is not positioned at its grid coordinates but
**aligned within the list's own rectangle**, near/centre/far on each axis. The grid
background, highlighter and blocker each collapse to a single draw.

**Notes** — this is the single-item display: the weapon slot, the detector slot, the upgrade
bench's subject. One widget serves both a hundred-cell backpack and a one-item pedestal, and
the mode switch is the difference. The alignment is authored as a word in the layout and read
by **searching the word for a letter** — a vertical alignment containing `t` is top, `b` is
bottom, anything else centre; horizontal `l` is left, `r` is right, else centre. Substring
matching, not equality, so an unrecognised word quietly becomes centre and a word that
happens to contain the letter is misread. A rebuild uses an enumeration and rejects unknown
words.

## Scrolling

**Contract** — the scroll bar appears when the board is taller than the viewport, or always
if the layout demands it. Its range is the overflow in canvas units, its step is a third of a
cell height, and scrolling moves the board's origin upward by the scroll position. A wheel
notch applies four steps — so one notch is one and a third cells.

**Notes** — resetting the scroll range always returns the board to the top. Any capacity or
cell-size change therefore scrolls the player back to the top of their inventory, which is
the original's behaviour and is why capacity is not adjusted while a screen is open.

## The condition indicator

**Contract** — when a condition bar is attached, each update sets it from the condition of
the item **at index zero**, quantized upward to fifteenths; an empty list reads zero.

**Notes** — index zero, not the selected item: this exists for the single-item slots, where
there is only ever one. The fifteen-step quantization matches the number of distinct segments
in the shipped bar texture, so the bar always lands on a segment boundary instead of clipping
one mid-way.

## Decoration attachment

**Contract** — the highlighter, blocker and condition bar are widgets the *screen* builds
from its own layout and then hands over. On attachment the list adopts any that has no
parent, takes ownership of it, and **rebases its position from absolute into the list's own
coordinates**.

**Notes** — the rebase is not a convenience. Chapter 15 clips against a frustum derived from
the ancestor chain; a widget authored at absolute coordinates and then adopted would be
measured against its new parent's origin and land outside the visible region. Converting once
at attachment is the fix. The flag that suppresses it exists for callers that already authored
the position relative to the list.

## `ClearAll` / ownership

**Contract** — clearing releases every cell, detaches every item, and optionally destroys
items and their stack children. Items are explicitly *not* auto-deleting: a list never owns
its items, because the same item widget is handed back and forth between lists during a
drag. Destruction is the caller's decision, expressed as a flag.
