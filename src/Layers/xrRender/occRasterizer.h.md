# src/Layers/xrRender/occRasterizer.h

> The dimensions, the fixed-point depth encoding and the buffer set of the software occlusion rasterizer.

**Needs** — [`occRasterizer.cpp`](occRasterizer.cpp.md) · [`occRasterizer_core.cpp`](occRasterizer_core.cpp.md) · [`HOM.h`](HOM.h.md)
**Used by** — [`HOM.cpp`](HOM.cpp.md) · [`HOM.h`](HOM.h.md) · [`occRasterizer.cpp`](occRasterizer.cpp.md) · [`occRasterizer_core.cpp`](occRasterizer_core.cpp.md)
**Tier floor** — T1: it fixes the exact size and element type of five fixed-size depth buffers that are scanned linearly, and a fixed-point encoding chosen so a comparison is one integer compare.

## Purpose

The engine culls against a *hierarchical occlusion map*: a tiny depth image of the world's big blockers, rasterized on the processor each frame, against which whole objects are tested with a rectangle-and-depth query. This header declares the shape of that image. The rasterization is in [`occRasterizer_core.cpp`](occRasterizer_core.cpp.md); the pyramid build and the query are in [`occRasterizer.cpp`](occRasterizer.cpp.md); the choice of which geometry to feed it is in [`HOM.cpp`](HOM.cpp.md).

## Dimensions

```text
base_size   = 64            # the occlusion image is 64 x 64, full screen
level_sizes = 64, 32, 16, 8 # four pyramid levels, each half the previous
padded_size = base_size + 4 # the working buffers are 68 x 68
```

The two-pixel border on every side of the working buffers is the decision worth recording. The scan converter writes a pixel and then looks at its neighbours one and two rows away to fill cracks between adjacent triangles; the border is what lets it do that at the image edge without a bounds test in the inner loop. Every coordinate is therefore biased by two on entry to the rasterizer and unbiased on the way out, and *only the inner sixty-four by sixty-four is real*.

Sixty-four by sixty-four for a whole screen is coarse on purpose: the map is a conservative *rejection* test, and every pixel it has costs a scan-line write. A rebuild is free to choose a different size, but the border must scale with the crack-filling reach, and the pyramid must remain a power-of-two chain.

## Depth encoding

```text
depth_scale = 2^30          # maps the clip-space depth range [-2, 2] onto a signed integer
depth_type  = int (32-bit, signed)
```

The working buffer holds real depths; the pyramid holds this fixed-point form. Converting once at the end of rasterization means the query — which reads many pyramid pixels per object — compares integers, and integer comparison has no denormal, no NaN and no ordering surprise. The range is `[-2, 2]` rather than `[0, 1]` because the rasterizer works in a clip space that has not been divided down and can overshoot; depths are clamped just short of the bounds before conversion, so a value exactly at the limit cannot overflow the encoding.

There is a second, sixteen-bit encoding declared alongside, with its own scale of `2^14 − 1`. Nothing uses it. It is a vestige of an abandoned attempt to halve the pyramid's memory.

## State

```text
RECORD Rasterizer
  coverage      : grid[padded_size][padded_size] of optional<Triangle>
  depth         : grid[padded_size][padded_size] of real
  pyramid[0..3] : grid[level_sizes[i]][level_sizes[i]] of int (32-bit)
```

Invariants:

- `coverage` and `depth` are parallel: a pixel's coverage entry names the triangle that wrote its depth. The crack filler needs that identity — it only fills between pixels belonging to the *same or adjacent* triangles — which is why the rasterizer carries a whole second grid of pointers rather than depth alone.
- Pyramid level zero is the converted inner region of `depth`; each further level is the **maximum** of its four children. Maximum, not minimum: the test asks "is this object behind everything here", so the conservative summary of a block is its *farthest* pixel.
- One instance exists per process. It is not re-entrant, and the frame that rasterizes into it must complete before the frame that queries it begins.

## `Triangle`

```text
RECORD Triangle
  adjacent  : list<Triangle> of exactly 3   # the triangles sharing each edge, or none
  raster    : list<vector3> of exactly 3    # vertices already in rasterizer pixel space
  plane     : plane
  area      : real
  flags     : int
  skip      : int
  centre    : vector3
```

`adjacent` is what makes crack filling safe: two pixels may be bridged only when their triangles are the same triangle or share an edge. That adjacency is built once, when the occlusion geometry is loaded, and is the reason the occluder set is a *mesh* rather than a triangle soup.

## Exported units

- **`Rasterizer`** (`occRasterizer`) — the buffers above plus `clear`, `rasterize`, `propagate` and `test`, described in [`occRasterizer.cpp`](occRasterizer.cpp.md) and [`occRasterizer_core.cpp`](occRasterizer_core.cpp.md), together with the fixed-point conversions and raw accessors to the three buffer families. The accessors exist because the scan converter is a set of free functions reaching into the buffers without going through the object, which is a performance shape, not a design.
- **`Raster`** — the single process-wide instance.
- **The debug pixel-box array** — in a debug build, the rasterizer also keeps one world-space box per occlusion pixel so the map can be drawn back into the scene. It is a debugging aid, not part of the algorithm.
