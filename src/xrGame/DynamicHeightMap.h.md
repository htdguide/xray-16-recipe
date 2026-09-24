# src/xrGame/DynamicHeightMap.h

> Declares the camera-following ground-height cache implemented in [`DynamicHeightMap.cpp`](DynamicHeightMap.cpp.md), and fixes its dimensions as constants.

**Needs** — [`DynamicHeightMap.cpp`](DynamicHeightMap.cpp.md)
**Used by** — [`DynamicHeightMap.cpp`](DynamicHeightMap.cpp.md)
**Tier floor** — T3: a declaration plus five constants

## Purpose

Declares the three layers of the height cache and, more usefully, pins its geometry. The
constants are the substance of the header and a rebuild needs them:

```text
slot_size    = 4 metres          # one cache slot's edge
margin       = 4 slots           # how far the window extends each way from the centre
window       = 9 x 9 slots       # 2 * margin + 1; odd so that "centred" is exact
precision    = 16 x 16 samples   # per slot, so 0.25 m between samples
```

Coverage is therefore thirty-six metres square at quarter-metre resolution. Substance in
[`DynamicHeightMap.cpp`](DynamicHeightMap.cpp.md).

Exported units:

- `CHM_Static` — the scrolling cache over the level's static collision geometry, its slot
  pool, its window of references and its rebuild queue.
- `CHM_Dynamic` — the intended layer for moving surfaces; unimplemented.
- `CHeightMap` — the public face: refresh both layers once per frame, answer with the
  greater height.
- `Query(x, z)` — the cached height at a horizontal position.
- `Query(position, direction)` — a declared three-dimensional ray query with no
  implementation anywhere in the tree.
