# src/Layers/xrRender/DetailManager_Decompress.cpp

> Turns one cell's four density maps into actual plants: dither the density to decide where, ray-cast down to find the ground, randomize yaw and scale from a seed derived from the cell's own coordinates.

**Needs** — [`DetailManager.h`](DetailManager.h.md) · [`DetailFormat.h`](DetailFormat.h.md) · [`xrCDB/xrCDB.h`](../../xrCDB/xrCDB.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../../xrMaterialSystem/GameMtlLib.h.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it is arithmetic and collision queries, with no device contact at all. What holds it at T2 rather than T3 is the cost — a few thousand ray-triangle tests per cell, seven cells a frame — and the requirement that the result be bit-identical run to run.

## Purpose

The single most interesting file in the grass layer. The shipped data contains no plants; it contains, per 2-metre cell, four density maps of four values each. This file is the function that turns four bytes of density into thirty or eighty individually placed, rotated, scaled and lit plants, and it must produce **exactly the same plants every time** — otherwise the grass would shimmer and rearrange as the camera moved away and back.

## Determinism — the central constraint

```text
seed = 0x12071980 XOR (cell_x * cell_z)

Four independent generators are seeded identically from it:
  one for which model is chosen at a grid point
  one for the position jitter and the dither phase
  one for the yaw
  one for the scale
```

**Invariants**

- The seed depends only on the cell's world coordinates. A cell decompressed, evicted and decompressed again produces byte-identical plants. This is not a nicety: the cache evicts and refills constantly as the player walks, and any non-determinism would be visible as grass popping into different positions.
- The seed uses the **product** of the coordinates, and the original says why: to break up the rows. A sum, or either coordinate alone, gives neighbouring cells correlated seeds and the grass visibly lines up in bands. The product decorrelates them cheaply. It also means the cells along either axis at zero all share a seed — a real flaw that shows as a stripe of identical cells through the world origin, which no shipped level places the player near.
- The constant is a date. It is a signature, not a value; any constant works.
- **Four separate generators, not one.** Each decision draws from its own stream, so adding a decision to one — a new random property on a plant — does not shift every subsequent decision and rearrange the entire level's grass. That is a maintenance property, and it is the reason a rebuild should keep the streams separate rather than sharing one.
- One draw escapes this discipline: the choice of which of the two wind phases a plant belongs to comes from the **global** random source, not a cell-seeded one. That makes the wind assignment non-deterministic across evictions. It is invisible because the two phases differ only in timing, and it looks like an oversight rather than a decision.

## `decompress(cell)`

**Contract** — fills one cell's plant lists from its density record. Queries the level's static collision database. Allocates plants from the manager's pool. Runs on the background task, one cell at a time; seven per frame at most. Returns immediately for an empty cell.

```text
FUNCTION decompress(cell)
  cell.state = ready
  IF cell.empty  RETURN

  record = the file's record for this cell
  triangles = static_collision.box_query(cell's provisional box)
  IF no triangles  RETURN            # grass needs ground to stand on

  FOR EACH of the four models
    expand its 4-bit corner densities to a 0..255 range

  density  = the detail-density setting
  jitter   = density / 1.7
  steps    = ceil(2.0 / density)     # candidate points per axis across the cell
  bounds   = empty

  FOR z FROM 0 TO steps, FOR x FROM 0 TO steps
    dither_phase_x, dither_phase_z = two draws from the jitter stream
    candidates = the models whose dithered density at (x, z) exceeds the
                 dither threshold at (x + phase_x, z + phase_z) modulo 16
    IF no candidates  CONTINUE
    model = one candidate, drawn from the selection stream

    position.xz = the grid point, plus a jitter draw in ±jitter
    position.y  = drop_to_ground(position, triangles)      # see below
    IF no ground was found beneath the cell's floor  CONTINUE

    scale  = a draw in [model.min_scale * 0.5, model.max_scale * 0.9] * height setting
    yaw    = a draw in [0, full turn)
    lighting = the cell's baked sun and sky values, dequantized
    wind class = still if the model forbids waving, else one of two phases
                 (a quarter chance of the second)

    append the plant; merge its transformed bounds into `bounds`

  cell.bounds = bounds          # replace the provisional box with the real one
```

**Invariants**

- **The candidate grid is derived from the density setting, not the cell.** Turning detail density down widens the spacing and produces fewer candidate points; it does not thin an existing set. This is why the density setting changes the grass *layout*, not just its count — a consequence a rebuild should be aware of before "fixing" it.
- **The jitter is the spacing divided by 1.7.** The number has no derivation anywhere. It is close to the spacing over the square root of three, which would be the radius at which jittered points on a square grid stop forming visible rows without overlapping; whether that was the reasoning is unrecoverable.
- **The scale range is not the authored range.** The minimum is multiplied by 0.5 and the maximum by 0.9. The authoring tools show the author one range and the engine uses a wider, lower one. There is no comment and no recoverable reason; it reads like a late global tweak to make the grass shorter and more varied, applied here rather than in the data because the data had already shipped.
- **The lighting is per cell, not per plant.** Every plant in a 2-metre cell gets the same baked sun and sky values. At grass scale this is invisible and it is why the cell record carries only one colour.
- The bounds written back are the *union of the plants' transformed bounding boxes*, which is almost always much tighter than the provisional box — the file's height range covers the terrain under the cell, and the plants occupy a few tens of centimetres of it. Tightening the box is what makes the coarse visibility test worth having.

## `drop_to_ground(position, triangles)`

**Contract** — finds the highest surface beneath a candidate point, ignoring surfaces the material system marks as passable. Returns nothing usable when there is no such surface above the cell's floor.

```text
FUNCTION drop_to_ground(candidate, triangles)
  best = cell floor - 5                       # a sentinel below any valid answer
  cast a ray straight down from the cell's ceiling
  FOR EACH triangle IN triangles
    IF its surface material is marked passable   CONTINUE
    IF the ray hits it at a non-negative distance
      best = max(best, ceiling - distance)
  RETURN best
```

**Invariants**

- The ray starts at the cell's *ceiling* — the top of the file's recorded height range — and the answer is the **highest** hit, not the first. Grass grows on the topmost surface: on a bridge, not on the ground under it.
- **Passable surfaces are skipped.** The material system marks some surfaces as walk-through (foliage volumes, grass-height markers, some water). Grass must not grow on the top of a bush.
- A candidate whose highest hit is below the cell's floor is discarded rather than clamped. The cell's recorded floor is authoritative; a hit below it means the ray found geometry belonging to a different part of the world, seen through a gap.
- The scan is linear over every triangle the box query returned, for every candidate point — a few dozen triangles times a few thousand candidates per cell. This is the whole cost of decompression and the sole reason for the seven-cells-per-frame budget. A rebuild with a faster ground query (a rasterized height field per cell, built once) could raise that budget substantially, and it is the obvious place to spend optimization effort.

## The dither test

**Contract** — is there a plant of this model at this grid point?

```text
FUNCTION dithered(density_corners, x, z, phase_x, phase_z, steps)
  value = interpolate(density_corners, x, z, steps)     # the symmetric bilinear
  RETURN round(value) > threshold_matrix[(x + phase_x) MOD 16][(z + phase_z) MOD 16]
```

**Invariants** — The phase offsets are redrawn **per grid point**, not per cell. That is what stops the threshold matrix's own 16×16 period from appearing as a texture in the grass. It also means the test is not a true ordered dither any more — it is an ordered dither with a random phase, which is closer to blue noise, and is a better answer than either alone. A rebuild may substitute any blue-noise mask; it will not produce the original's exact placement.

Note that the interpolation is the symmetric average-of-two-bilinears described in [`DetailFormat.h`](DetailFormat.h.md), not ordinary bilinear interpolation.

## What could not be recovered

- The 1.7 jitter divisor.
- The 0.5 and 0.9 scale multipliers.
- Why the wind-phase draw uses the global generator while everything else uses a cell-seeded one.
