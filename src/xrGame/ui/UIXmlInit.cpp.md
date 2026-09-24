# src/xrGame/ui/UIXmlInit.cpp

> The four layout readers the game layer adds to the toolkit's element vocabulary — above all the
> inventory grid's, which is where every placement rule enters from data.

**Needs** — [`UIXmlInit.h`](UIXmlInit.h.md) · [`UIDragDropListEx.h`](UIDragDropListEx.h.md) · [`UIDragDropReferenceList.h`](UIDragDropReferenceList.h.md) · [`UITabButtonMP.h`](UITabButtonMP.h.md) · [`UISleepStatic.h`](UISleepStatic.h.md) · [`UILabel.h`](UILabel.h.md) · [`xrUICore/XML/UITextureMaster.h`](../../xrUICore/XML/UITextureMaster.h.md) · [`xrUICore/Windows/UIFrameWindow.h`](../../xrUICore/Windows/UIFrameWindow.h.md) · [`xrUICore/Lines/UILines.h`](../../xrUICore/Lines/UILines.h.md)
**Used by** — [`UIXmlInit.h`](UIXmlInit.h.md)
**Tier floor** — T3: attribute reading into widget configuration

## Purpose

Chapter 15 holds that a layout document may configure a widget type the engine already knows but may
never introduce one, and that the element vocabulary is therefore exactly the set of reader
functions. This file is the game layer's four-entry extension to that set.

Only one of the four carries a real decision: the inventory grid's reader is where **every**
behaviour of the inventory and trade grids — cell size, spacing, whether items stack, whether the
player may place items freely, how virtual cells align — arrives from data. The other three are a
handful of attributes each.

## State

`Stateless.` Readers configure a widget the caller already built.

## `InitDragDropListEx`

**Contract** — Configures an inventory grid. Fails hard if the named element is missing, because a
screen that expects a grid and gets none cannot recover. Reads position, size and alignment, then
the grid geometry, then eight behaviour flags.

```text
FUNCTION InitDragDropListEx(document, path, index, grid)
  REQUIRE document has path[index]
  position <- (x, y); size <- (width, height)
  apply the alignment mode to the position
  grid.place(position, size)

  grid.cell_size    <- (cell_width, cell_height)
  grid.cell_spacing <- (cell_sp_x, cell_sp_y)
  grid.capacity     <- (cols_num, rows_num)          # the starting capacity, not a cap

  grid.auto_grow          <- attribute "unlimited"              (default off)
  grid.group_identical    <- attribute "group_similar"          (default off)
  grid.free_placement     <- attribute "custom_placement"       (DEFAULT ON)
  grid.vertical_placement <- attribute "vertical_placement"     (default off)
  grid.always_show_scroll <- attribute "always_show_scroll"     (default off)
  grid.show_condition_bar <- attribute "condition_progress_bar" (default off)
  grid.virtual_cells      <- attribute "virtual_cells"          (default off)
  IF virtual cells THEN
    grid.cell_vertical_alignment   <- attribute "vc_vert_align"
    grid.cell_horizontal_alignment <- attribute "vc_horiz_align"
  grid.background_colour <- the element's colour (default opaque white)
```

**Invariants** — `custom_placement` defaults **on** while every other flag defaults off. That
asymmetry is load-bearing: free placement — the player may drop an item into any empty cell rather
than the grid packing it — is the inventory's normal behaviour, and a grid that wants packing must
say so. A rebuild defaulting it off silently turns every shipped inventory into a packed list.

The row and column counts are a *starting* capacity, not a limit; combined with `unlimited` the grid
grows past them. The two alignment attributes are read only in virtual-cell mode and are read as
free-form strings the grid parses itself, so the vocabulary of alignment names lives with the grid.

**Notes** — The two colour-and-size defaults, and every attribute name above, are frozen: they
appear in shipped layout documents for all three games.

## `InitTabButtonMP`

**Contract** — Configures a multiplayer tab button. Runs the base three-state button reader first,
then reads two optional groups:

- an `idention` child supplying the two text offsets — `over_x`/`over_y` for the hovered position
  and `normal_x`/`normal_y` for the resting one, all defaulting to zero;
- a `hint` child, whose *presence* is what causes the button's secondary label to be created at all,
  configured as a plain picture-and-text widget.

See [`CUITabButtonMP`](UITabButtonMP.cpp.md) for what the button does with them.

**Notes** — This reader writes the button's fields directly rather than through setters, which is
why those fields are public. The two are one unit split across files.

## `InitSleepStatic`

**Contract** — Fails hard if the element is missing, then configures the widget with the plain
picture reader. It adds nothing of its own — it exists so that `sleep_static` is a *name* in the
element vocabulary, which is what lets a layout declare one at all.

## `InitHintWindow`

**Contract** — Configures a window that carries a delayed tooltip: the base window reader, then the
element's text content as a **localization identifier** for the tooltip (defaulting to the literal
`no hint`, which is what an unconfigured tooltip visibly shows), then the dwell delay from the
`delay` attribute.

**Notes** — The default tooltip text is a literal rather than an identifier, so an element that
forgets its text shows the words `no hint` rather than a missing-string marker. That is deliberate:
it reads as a developer oversight rather than as a broken localization.
