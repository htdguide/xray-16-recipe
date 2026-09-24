# src/Layers/xrRender/occRasterizer.cpp

> Turns the rasterized depth image into a maximum-pyramid, and answers "is this rectangle entirely behind what has been drawn".

**Needs** — [`occRasterizer.h`](occRasterizer.h.md) · [`occRasterizer_core.cpp`](occRasterizer_core.cpp.md) · [`xrRender_console.h`](xrRender_console.h.md) · [`R_Backend.h`](R_Backend.h.md)
**Used by** — [`occRasterizer.h`](occRasterizer.h.md) · [`occRasterizer_core.cpp`](occRasterizer_core.cpp.md)
**Tier floor** — T1: the pyramid build and the query are tight scans over fixed-size integer grids, and the query is on the critical path of every visibility decision in the frame.

## Purpose

Two halves of the occlusion map that are not scan conversion: the per-frame *finish* step that closes remaining cracks and builds the pyramid, and the *query* that everything else in the renderer calls.

## `clear`

**Contract** — Reset the working buffers for a new frame: no coverage anywhere, depth everywhere at the far value of one. Touches the padded buffers in full, border included, because the crack filler reads the border.

## `propagate`

**Contract** — Called once after every occluder has been rasterized, before any query. Performs a final vertical crack fill over the real image, converts it to fixed point as pyramid level zero, and reduces that to the three coarser levels. Reads and writes only this object.

```text
FUNCTION propagate()
  FOR EACH pixel (x, y) IN the real 64 x 64 region
    p = padded index of (x, y)

    # Vertical crack fill. The scan converter closes horizontal gaps as it
    # goes; a gap one or two scan lines tall between two pieces of the same
    # surface survives it. Here, a pixel is filled from the pixel one row up
    # when that pixel's triangle also covers one or two rows DOWN — which is
    # the signature of a crack rather than a genuine silhouette edge.
    above = coverage[p - one_row]
    IF above exists
      IF above shares an edge with coverage[p + one_row]
        interpolated = mean(depth[p - one_row], depth[p + one_row])
      ELSE IF above shares an edge with coverage[p + two_rows]
        interpolated = mean(depth[p - one_row], depth[p + two_rows])
      ELSE
        interpolated = none
      # Only ever bring a pixel NEARER. The map must stay conservative: a
      # filled crack that pushed depth farther would hide real geometry.
      IF interpolated exists AND interpolated < depth[p]
        coverage[p] = above ; depth[p] = interpolated

    # Convert to fixed point, clamped just inside the encodable range.
    pyramid[0][y][x] = to_fixed(clamp(depth[p], -1.99, +1.99))

  pyramid[1] = reduce_max(pyramid[0])
  pyramid[2] = reduce_max(pyramid[1])
  pyramid[3] = reduce_max(pyramid[2])
```

**Invariants**

- Reduction takes the **maximum** of each two-by-two block. The query asks whether an object is *behind* the map, so a block's conservative summary is its farthest pixel: if the object is behind even that, it is behind all four.
- Every fill may only decrease a depth. This is the map's safety property — the depths it holds must never be farther than the real geometry, or it will cull something visible.
- The crack fill's "one or two rows" reach is exactly why the working buffers carry a two-pixel border.

## `test`

**Contract** — Given a rectangle in normalised screen coordinates and a nearest depth, report whether *any* pixel of the map in that rectangle is farther than the depth — that is, whether the object might be visible. Returns visible on any doubt. Reads only; safe to call from several threads once `propagate` has finished.

```text
FUNCTION test(x0, y0, x1, y1, z) -> bool
  # Round the depth AWAY from the viewer and add one unit. Both are deliberate
  # slack: a rounding error in the object's own depth must never make it
  # invisible, so the comparison is biased towards "visible".
  z_fixed = to_fixed_round_up(z) + 1
  RETURN test_level(pyramid[0], 64, x0, y0, x1, y1, z_fixed)

FUNCTION test_level(level, size, x0, y0, x1, y1, z) -> bool
  # Map the normalised rectangle to pixel indices, rounding to the NEAREST
  # pixel centre, then clamp. The clamp of the far edge is against the near
  # edge, not against zero, so a rectangle entirely off one side collapses to
  # a single edge pixel rather than inverting.
  ix0 = clamp(floor(x0 * size + 0.5), 0, size - 1)
  ix1 = clamp(floor(x1 * size + 0.5), ix0, size - 1)
  iy0 = clamp(floor(y0 * size + 0.5), 0, size - 1)
  iy1 = clamp(floor(y1 * size + 0.5), iy0, size - 1)

  FOR EACH pixel IN the index rectangle
    IF z < pixel THEN RETURN visible     # something here is farther: not occluded
  RETURN occluded
```

**Notes** — The pyramid is built but the query uses **only level zero**. The original contains the obvious hierarchical form — test a coarse level, and descend to level zero only where it says "visible" — commented out. At sixty-four by sixty-four an object's rectangle covers few enough pixels that the descent's branch cost outweighed the pixels it saved. The coarser levels are dead weight in the shipping configuration; a rebuild may drop them, and should reinstate them only if it also raises the map's resolution.

The early return on the first farther pixel means the loop is fast for visible objects and slow for occluded ones — the opposite of what one would want, but unavoidable: proving occlusion requires reading every pixel.

## `debug_draw`

**Contract** — In a debug build, and only while the corresponding display flag is set, reconstruct each occlusion pixel as a world-space box at its recorded depth and draw it as a wireframe, coloured from green at the near plane to red at the far. The reconstruction inverts the projection matrix analytically per pixel; the boxes are captured once and then redrawn until the flag is cleared, so the map can be inspected from a moved camera. Purely a diagnostic; absent from a shipping build.
