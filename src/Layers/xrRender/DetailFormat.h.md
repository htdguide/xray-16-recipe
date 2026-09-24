# src/Layers/xrRender/DetailFormat.h

> The frozen on-disk format of the grass layer: a 2-metre grid over the level, each cell naming up to four models and carrying a quantized density map and baked lighting in sixteen bytes.

**Needs** — _(none beyond the core types)_
**Used by** — [`DetailManager.h`](DetailManager.h.md) · [`DetailManager_CACHE.cpp`](DetailManager_CACHE.cpp.md) · [`DetailManager_Decompress.cpp`](DetailManager_Decompress.cpp.md) · [`DetailModel.cpp`](DetailModel.cpp.md)
**Tier floor** — T1: the slot record is a bit-packed memory image read directly out of a mapped file region; every field's width and position is part of the shipped data.

## Purpose

Defines the format of a level's `level.details` file, which is what the grass and debris layer is generated from. Nothing here computes anything; it is the file's shape plus the quantization rules that turn its integers back into world values. **Frozen** — this format ships with all three games.

The central design decision is that the grass is not *stored*, it is *regenerated*. The file holds a coarse grid of density maps, and the engine deterministically synthesizes individual plants from them at run time (see [`DetailManager_Decompress.cpp`](DetailManager_Decompress.cpp.md)). This is why a level's grass costs a few hundred kilobytes rather than a few hundred megabytes.

## The grid

```text
DETAIL_VERSION   = 3        # the only version the engine accepts; older levels are rejected outright
DETAIL_SLOT_SIZE = 2.0      # metres; one cell of the grid
```

```text
RECORD DetailHeader
  version      : int (32-bit)     # must equal 3
  object_count : int (32-bit)     # at most 64; see the six-bit model id below
  offs_x       : int (32-bit, signed)   # grid origin, in cells, relative to world zero
  offs_z       : int (32-bit, signed)
  size_x       : int (32-bit)     # grid extent in cells
  size_z       : int (32-bit)
```

**Invariants**

- The grid is indexed row-major by z then x: `index = z * size_x + x`. The header's own index helper recomputes the inverse and asserts it matches, which is a cheap guard against an off-by-one in the level compiler and costs nothing at load.
- A cell's *centre* in world space is `(cell - offs) * 2.0 + 1.0`; its bounds are that centre ± 1.0, inset by a small epsilon on each side so that adjacent cells' rectangles do not overlap at the seam. Grass is assigned to exactly one cell, and an overlapping boundary would double-populate the seam.
- The mapping from a camera position to a cell is `floor(position / 2.0 + 0.5)` — note the half-cell bias, which centres the grid on the camera's cell rather than putting the camera at a corner. The whole cache is built around the camera being at the centre cell.
- A query outside the grid's extent returns a synthetic empty cell rather than failing. Levels are not rectangular and the grid is their bounding box; most of it is out of bounds or empty.

## The slot record — sixteen bytes per cell

```text
RECORD DetailSlot                       # exactly 16 bytes, bit-packed
  y_base    : int (12-bit)   # ground height: 1 unit = 20 cm, biased by -200 m
                             #   range -200.0 m .. +619.2 m
  y_height  : int (8-bit)    # vertical extent above y_base: 1 unit = 10 cm, 0 .. 25.6 m
  id0..id3  : int (6-bit each)   # four model ids into the file's object list; 63 means empty
  c_dir     : int (4-bit)    # baked sun contribution at this cell, 0..1 quantized over 15
  c_hemi    : int (4-bit)    # baked ambient/sky contribution
  c_r,c_g,c_b : int (4-bit each)  # baked colour, 4.4.4
  palette   : list<DetailPalette> of length 4     # 8 bytes: see below

RECORD DetailPalette             # 2 bytes: the density of ONE model at the cell's four corners
  a0, a1, a2, a3 : int (4-bit)
```

**Invariants**

- **Sixteen distinct model ids are not enough; sixty-four is.** The id field is six bits and the sentinel for "no model" is all ones (63), so a level may define at most 63 distinct detail models and a cell may host at most four of them. Both limits are hard and the level compiler enforces them.
- The height quantization is *asymmetric on purpose*: the base is stored coarsely (20 cm) over a wide range because it only has to locate the ground, and the extent finely (10 cm) over a narrow range because it bounds the cell for culling. When writing, the residual error from quantizing the base is folded into the height and the height is rounded *up*, so the cell's box always contains the real geometry. Rounding the height down would let grass poke out of its own bounding box and disappear at the wrong moment.
- The palette is a density field, not a colour: each model's four values are the density at the cell's four corners, and the decompressor bilinearly interpolates between them. An all-zero palette means "this model is not actually present here" — and the format allows a non-empty id with an empty palette, so the engine normalizes by clearing the id whenever its palette is empty. That normalization must be done on load or the decompressor wastes work on models with zero density everywhere.
- A cell is empty when all four ids are the sentinel. This is checked constantly — it is the first test in every cache and visibility loop — which is why it is four comparisons against a constant rather than a stored flag.

## The four-corner density interpolation

The value at a point inside a cell is not a plain bilinear interpolation. It is the *average of the two axis-major bilinear interpolations*: interpolate along x then z, interpolate along z then x, and average the two. The two are not the same function on a non-planar four-corner patch, and averaging them produces a symmetric surface with no preferred axis.

**Notes** — Whether the asymmetry of ordinary bilinear interpolation was visible in the grass is unrecoverable. The averaging costs three extra multiply-adds per sample in a loop that runs a few thousand times per cell, so somebody thought it mattered. A rebuild that uses plain bilinear interpolation will produce *differently placed grass* than the original — not wrong, but not identical, which matters if visual conformance against the original is the acceptance test.

## `file layout`

**Contract** — the file is the engine's standard recursive chunked container with three top-level chunks:

```text
chunk 0 : the header record
chunk 1 : the models, each in a sub-chunk numbered by its id (see DetailModel.cpp)
chunk 2 : the slot array, size_x * size_z records of 16 bytes, row-major
```

**Invariants** — The slot array is **not copied on load**. The engine takes a pointer into the mapped file region and keeps the file open for the level's lifetime. At a couple of hundred thousand cells this is several megabytes it would otherwise duplicate, and the array is read-only. A rebuild that parses the file into its own structures must account for that memory; a rebuild that maps it must account for the file handle's lifetime.
