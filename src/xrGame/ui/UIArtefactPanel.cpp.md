# src/xrGame/ui/UIArtefactPanel.cpp

> The belt artefact strip: a row of icons drawn by reusing one primitive, with no widgets
> behind them.

**Needs** — [`UIArtefactPanel.h`](UIArtefactPanel.h.md) · [`UIInventoryUtilities.h`](UIInventoryUtilities.h.md) · [`UIXmlInit.h`](UIXmlInit.h.md)
**Used by** — [`UIArtefactPanel.h`](UIArtefactPanel.h.md)
**Tier floor** — T2: emits quads directly from a per-frame loop

## Purpose

The overlay must show what is on the belt, and the belt changes often. Building a widget per
artefact would mean constructing and destroying widgets on every belt change, in a path that
runs while the player is moving. Instead the panel keeps only the icons' *texture rectangles*
and draws them all with one reused primitive.

That is the file's one decision, and it is the reason this widget looks unlike every other
widget in the chapter.

## State

```text
RECORD ArtefactPanel
  cell_size : (real, real)   # authored reference cell, default 50 x 50
  scale     : real           # authored, default 0.5
  vertical  : bool           # lay the strip down instead of across
  indent    : int            # gap between icons, default 1
  rects     : list<rect>     # one atlas sub-rectangle per artefact on the belt
  item      : drawing primitive, reused for every icon
```

Invariant: `rects` is the whole model. It is rebuilt wholesale by `InitIcons` and read only by
`Draw`; nothing else refers to an artefact.

## `InitFromXML`

**Contract** — Configure the panel's rectangle as a window, then read four attributes: the
reference cell width and height (defaulting to 50 each), the icon scale (defaulting to 0.5)
and whether the strip runs vertically, plus the inter-icon gap.

## `InitIcons`

**Contract** — Rebuild the rectangle list from the artefacts currently on the belt. For each
artefact, read its four grid coordinates from its configuration section and convert them to an
atlas rectangle by multiplying by the icon grid's cell dimensions — the same 50-unit grid that
governs every inventory icon (see [`UICellCustomItems.cpp`](UICellCustomItems.cpp.md)).

```text
FUNCTION init_icons(artefacts)
  bind the primitive to the shared equipment icon atlas
  rects = empty
  FOR EACH artefact
    s = artefact's configuration section
    r.top_left     = (s.inv_grid_x, s.inv_grid_y) * grid cell size
    r.bottom_right = (s.inv_grid_width, s.inv_grid_height) * grid cell size + r.top_left
    append r
```

**Notes** — The configuration stores width and height, not a bottom-right corner; the addition
at the end is the conversion. Every icon reader in the chapter does this and each does it
slightly differently, which is a maintenance hazard rather than a decision.

## `Draw`

**Contract** — Walk the rectangle list, placing and drawing the single primitive once per
entry, advancing along the strip's axis.

```text
FUNCTION draw()
  x, y = this panel's absolute top-left
  aspect = cell_size.x / cell_size.y

  FOR EACH r IN rects
    # The two axes are scaled differently and CROSSED: the drawn width
    # comes from the rectangle's height and the drawn height from its
    # width, with the cell aspect applied to the second.
    size = (scale * r.height, aspect * scale * r.width)

    primitive.texture_rect = r
    primitive.size = size
    primitive.position = (x, y)

    IF horizontal THEN x = x + indent + size.x
    ELSE               y = y + indent + size.y

    primitive.render()

  draw children as usual
```

**Invariants** — The crossed axes are in the original and are almost certainly a slip: the
width should come from the rectangle's width. It survives because the shipped artefact icons
are square, which makes the two indistinguishable. A rebuild should use the matching axes,
and should expect the panel to look identical against shipped data either way.

The advance happens **before** the draw, so the first icon is drawn one gap in from the
panel's left edge rather than flush. That, too, is invisible with the shipped gap of 1.
