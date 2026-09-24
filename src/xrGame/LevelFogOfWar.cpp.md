# src/xrGame/LevelFogOfWar.cpp

> Per-level record of which parts of the map the player has walked near, and the pass that draws the unexplored parts over the map screen.

**Needs** — [`LevelFogOfWar.h`](LevelFogOfWar.h.md) · [`Level.h`](Level.h.md) · [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) · [`ui/UIMap.h`](ui/UIMap.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: builds a vertex buffer in place and hands it to the graphics device

## Purpose

Exploration state, per level, single player only. The world's horizontal extent is divided
into a fixed grid of cells; walking near a cell opens it permanently; the map screen draws
an opaque tile over every cell still closed. The state lives in the alife registry so it
survives leaving and re-entering the level, and is written into the save.

The drawing half and the state half are in one file because the grid's geometry is the same
question in both: the mapping between world coordinates and cell indices.

## State

```text
RECORD LevelFogOfWar
  level_name : text
  level_rect : rectangle in world coordinates (x and z; the map is a plan view)
  row_count  : int
  col_count  : int
  cells      : list<bool>    # row-major, row_count * col_count entries

RECORD FogOfWarManager
  registry : map<level, list<LevelFogOfWar>>   # in the alife registry, so it is saved
```

**Invariants**
- The cell size is fixed at 50 world units and the grid is derived from it, so `cells` has
  exactly `row_count * col_count` entries and the counts are a function of the level's
  bounds alone. Changing the cell size invalidates every existing save.
- Rows count **downward** from the top of the rectangle while world coordinates count
  upward, so the row index is computed against the rectangle's height rather than directly.
  That inversion is the map screen's convention and it is baked into the saved layout.
- The whole subsystem is inert outside single player: the lookup returns nothing, and the
  map screen draws no fog.

## `CFogOfWarMngr::GetFogOfWar`

**Contract** — the exploration record for a named level, creating and initializing one on
first request. Returns nothing outside single player. Lookup is a linear scan by level name
over the registry's list, which is fine because the list is one entry per visited level.

## `CLevelFogOfWar::Init`

**Contract** — sizes the grid from the level's authored bounds and allocates it closed, then
acquires the material pass and geometry stream used to draw it.

```text
FUNCTION init(level_name)
  bounds = game config [level_name]."bound_rect",
           default (-10000, -10000, 10000, 10000)
  # rounding is to the NEAREST cell, not down: a level whose extent is not a
  # whole number of cells still gets a cell covering its edge
  row_count = floor((bounds.height + cell_size / 2) / cell_size)
  col_count = floor((bounds.width  + cell_size / 2) / cell_size)
  cells = row_count * col_count entries, all closed
  acquire the fog material pass and a dynamic geometry stream
```

**Notes** — the default bounds are a twenty-kilometre square and are explicitly a stopgap
for a level whose configuration omits the key. A level that falls back to them gets a grid
of 160,000 cells, almost all of which are outside the world. A rebuild should treat a
missing bound as an error.

## `CLevelFogOfWar::Open` (by world position)

**Contract** — opens every cell overlapping a small square centred on a world position.
Positions outside the level's bounds are ignored rather than clamped — the player can stand
outside the authored rectangle and must not open a cell by wrapping.

```text
FUNCTION open(position)
  IF the grid is empty OR position is outside level_rect
    RETURN
  col = floor((position.x - rect.left) / cell_size)
  row = floor((rect.height - (position.y - rect.top)) / cell_size)   # inverted
  reach = ceil(open_radius / cell_size)
  target = the square of side 2*open_radius centred on position
  FOR EACH (r, c) IN the (2*reach+1) square around (row, col), skipping negatives
    IF the cell's world rectangle intersects target
      open(r, c)
```

**Notes** — the reveal radius is a quarter of the cell size, which makes the reach a single
ring and means standing anywhere opens the cell under you and any neighbour whose edge is
within a quarter cell. The effect is that walking a corridor opens a band roughly one cell
wide. Why a quarter and not a half is not recoverable; it is visibly a tuning choice.

Negative indices are skipped but indices past the far edge are not — they are caught by the
per-cell setter instead. The asymmetry is harmless and not a decision.

## `Draw`

**Contract** — draws the closed cells as an opaque grid over the map screen, clipped to the
visible part of the map and scaled by the map's current zoom. Only cells the view actually
covers are emitted, so the cost is proportional to what is on screen rather than to the
level's size.

```text
FUNCTION draw()
  map = the containing map screen
  visible = intersection of the map's clip rectangle with its own rectangle,
            expressed relative to the map's origin, then divided by the zoom
            and translated into world coordinates
  # grow by one cell so a partially visible edge cell is still drawn
  extend visible by one cell at the far edge
  cells = convert visible to cell indices
  origin_on_screen = the top-left of that cell block, in screen pixels

  cell_w = cell_size * zoom * horizontal interface scale
  cell_h = cell_size * zoom * vertical   interface scale

  lock a vertex range of 6 vertices per cell
  FOR EACH cell in the block
    pick its texture origin: one half of the material for open, the other for closed
    emit two triangles covering its screen rectangle, snapped to whole pixels
      and offset by half a pixel
  unlock; push the clip rectangle; draw; pop
```

**Invariants** — vertex positions are floored to integers and then shifted by half a pixel.
That is the classic texel-to-pixel alignment correction for this generation of graphics API;
without it the fog tiles bleed into each other at some zoom levels. A rebuild on an API with
different sampling rules must re-derive the offset rather than copy it.

**Notes** — open and closed are two halves of one material rather than two materials, chosen
by texture origin. That keeps the whole grid in a single draw with no state changes, which
is what makes drawing several thousand quads per frame acceptable.

Every cell in the visible block is emitted, open or closed, and the open ones sample a
transparent half. Emitting only closed cells would halve the geometry in a well-explored
level; the uniform loop was preferred.

## `GetTexUVLT`

**Contract** — the texture origin for one cell: the open half if the cell is open, the
closed half otherwise. An out-of-range index answers *closed*, so the border of a level
reads as unexplored rather than as an error.

## `ConvertRealToLocal` / `ConvertLocalToReal`

**Contract** — world coordinates to cell indices and back, for a point and for a rectangle.
Note these use the **uninverted** row convention, unlike the opening path — they are the
drawing path's mapping and the two must not be mixed.

**Notes** — having two different row conventions in one file, one for opening and one for
drawing, is the sharpest edge here. It works only because the saved grid is written and read
by the same pair. A rebuild should pick one convention and state it.

## `save` / `load`

**Contract** — writes the level name, the bounds, the two counts and the cell bits. Loading
reads the name, **re-initializes from configuration**, and then verifies that the stored
bounds and counts match what configuration now says before reading the bits.

**Invariants** — that verification is the format's integrity check: a save made against a
different version of the level's bounds is detected rather than silently misread. It is an
assertion, so in a release build a changed bound produces a misaligned grid instead of a
diagnostic. A rebuild should make it a real check.

**Notes** — the bounds and counts are written even though loading recomputes them from
configuration. They exist solely to be compared. That is a deliberate redundancy and worth
keeping.
