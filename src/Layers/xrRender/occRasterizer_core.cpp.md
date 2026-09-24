# src/Layers/xrRender/occRasterizer_core.cpp

> The software scan converter: one triangle becomes depth pixels, with the span ends widened by half a pixel and the gaps between neighbouring triangles bridged as it goes.

**Needs** — [`occRasterizer.h`](occRasterizer.h.md) · [`occRasterizer.cpp`](occRasterizer.cpp.md)
**Used by** — [`occRasterizer.cpp`](occRasterizer.cpp.md) · [`occRasterizer.h`](occRasterizer.h.md)
**Tier floor** — T1: a per-pixel inner loop over a fixed-size grid, written to run on the processor inside the frame budget; the whole file is a throughput argument.

## Purpose

The occlusion map's depths come from here. This is a depth-only triangle rasterizer with two properties an ordinary one does not have, and both exist because the map must be **conservative** — it may never claim a pixel is nearer than the geometry really is, and it must not leave holes that let a hidden object show through.

The two properties: spans are widened by half a pixel at each end and the widened parts are written only where they continue a neighbouring triangle; and after the triangle is laid down, its edges are walked and the pixels along them are patched from their vertical neighbours.

## Working state

The scan converter is a set of free functions over module-level working state, not methods. That is a performance shape — the inner loop reads the current triangle and the two buffers without an indirection — and a rebuild is free to make it an object.

```text
current_triangle : Triangle
pixels_written   : int          # returned as the triangle's coverage
vertex_low, vertex_mid, vertex_high : (x, y, z)   # sorted by y, biased into the padded grid
```

## `rasterize`

**Contract** — Draw one triangle into the working depth and coverage buffers, and report how many pixels it actually claimed. Reads only the triangle's already-projected vertices; writes only the two working grids. Does not allocate. Not re-entrant.

```text
FUNCTION rasterize(triangle) -> int
  current_triangle = triangle ; pixels_written = 0
  sort the three vertices by y into (low, mid, high)
  bias every x and y by the two-pixel border offset

  # A triangle is drawn as two sections split at the middle vertex. Which
  # section owns the scan line the middle vertex falls on depends on where in
  # that pixel it lands: past the pixel centre and the upper section takes it,
  # otherwise the lower one. Getting this wrong either double-writes a scan
  # line or leaves it blank.
  IF fractional_part(mid.y) > 0.5
    draw_section(upper, include_middle_line = true)
    draw_section(lower, include_middle_line = false)
  ELSE
    draw_section(upper, include_middle_line = false)
    draw_section(lower, include_middle_line = true)

  RETURN pixels_written
```

**Notes** — The triangle is *not* clipped against the image before rasterization. Clamping happens per scan line and per span, which is cheaper than a polygon clip for triangles that are mostly on screen, and the guard band of one pixel outside the real region absorbs the rest. A triangle that projects far off screen therefore costs scan lines that write nothing; the occluder selection upstream is what keeps that from mattering.

## `draw_section`

**Contract** — Rasterize the part of the triangle between two of its vertices' scan lines. Determines which of the two bounding edges is on the left by comparing their inverse slopes — and the comparison flips between the upper and lower section, because in the lower section the edges converge rather than diverge.

```text
FUNCTION draw_section(which, include_middle_line)
  choose the start and end scan lines from the sorted vertices, adjusting by
  one for the middle line, then clamp both into the grid
  RETURN IF the range is empty

  compute each bounding edge's dx/dy and dz/dy
  decide which edge is left from the sign of the slope comparison
  step both edges to the first scan line's exact y, including the
    sub-pixel offset introduced by rounding the start line

  # Half-pixel widening. Each span end carries the half-step its edge takes
  # per scan line; the scan converter is given both the nominal end and that
  # half-width, and decides per pixel what to do with the widened part.
  left_half  = left_dx  / 2 ; left_x  += left_half
  right_half = right_dx / 2 ; right_x += right_half

  FOR each scan line in the range
    scan(line, left_x, left_half, right_x, right_half, left_z, right_z)
    advance left_x, right_x, left_z, right_z by their per-line steps
```

## `scan` — one span

**Contract** — Write one scan line. This is the conservative part, and it is where the file's whole argument lives.

```text
FUNCTION scan(y, left_x, left_half, right_x, right_half, z_left, z_right)
  # Each end of the span has TWO positions: the outer one (widened by the
  # half-step) and the inner one. Between them is the "connector" region —
  # pixels the triangle partially covers.
  outer_left  = min(left_x  - left_half,  left_x  + left_half)
  inner_left  = max(left_x  - left_half,  left_x  + left_half)
  inner_right = min(right_x - right_half, right_x + right_half)
  outer_right = max(right_x - right_half, right_x + right_half)
  clamp all four into the guard-banded grid; RETURN IF empty

  interpolate z linearly from outer_left to outer_right

  # Push the whole span slightly FARTHER, by half of one pixel's depth step.
  # This places each pixel's depth at the far side of the surface it samples,
  # so an object standing flush against a wall from the outside is not clipped
  # by the wall's own occlusion pixel. It is the single most important
  # conservativeness adjustment in the file.
  z += 0.5 * abs(dz_per_pixel)

  # Left connector: partially covered pixels. Written ONLY where the pixel to
  # their left already belongs to this triangle or one sharing an edge with it
  # — i.e. where the coverage is genuinely continuous and the gap is an
  # artefact of two triangles meeting. The depth taken is the FARTHER of the
  # interpolated value and the neighbour's, again to stay conservative.
  FOR pixel FROM outer_left TO inner_left
    IF coverage[pixel - 1] shares an edge with current_triangle
       AND z < depth[pixel]
      write coverage and max(z, depth[pixel - 1])

  # Interior: unconditional nearest-wins.
  FOR pixel FROM inner_left TO inner_right
    IF z < depth[pixel] THEN write coverage and z

  # Right connector: the mirror of the left, walked inward from the outer end.
  FOR pixel FROM outer_right DOWN TO inner_right
    IF coverage[pixel + 1] shares an edge with current_triangle
       AND z < depth[pixel]
      write coverage and max(z, depth[pixel + 1])
```

**Invariants**

- A connector pixel is never written on adjacency alone: the interpolated depth must still win the depth test, and the depth actually stored is the *farther* of the two candidates. Both rules keep the bridge from pulling the surface nearer than it is.
- Adjacency is the triangle's precomputed edge neighbours plus identity. This is why the occluder geometry must be a connected mesh with adjacency built at load time; a triangle soup would bridge nothing and the map would be full of one-pixel seams.

## `patch_edges`

**Contract** — After a triangle is laid down, walk each of its three edges with an integer line stepper and, at each pixel on the edge and the two pixels directly above and below it, attempt a vertical bridge: if the pixel one row up and the pixel one row down belong to the same or adjacent triangles, fill the pixel between them with the mean of their depths — again only if that makes it nearer.

This is the vertical counterpart of the horizontal connectors, and it handles the seam where two triangles meet along a nearly horizontal edge. The final pass in [`occRasterizer.cpp`](occRasterizer.cpp.md) does the same thing once more over the whole image, catching seams whose two sides were rasterized at different times.

**Notes** — The edge walk is an integer error-accumulating line stepper chosen over floating point for exactness: the patch must visit precisely the pixels the scan converter touched, and a half-pixel disagreement between the two would leave the seam it was meant to close.

## Recovered but unused

The original also computes a barycentric-style "one third" constant and an alternative connector depth — a weighted mean of the interpolated and neighbouring depths rather than their maximum — which is present but commented out on both connectors. The maximum was chosen instead, consistent with conservativeness. A rebuild should use the maximum and need not carry the alternative.
