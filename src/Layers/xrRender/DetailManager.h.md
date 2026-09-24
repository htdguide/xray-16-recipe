# src/Layers/xrRender/DetailManager.h

> Declares the grass layer's whole machinery: the sliding cache of decompressed cells, the per-instance record, the two-level visibility grid and the two rendering paths.

**Needs** — [`DetailFormat.h`](DetailFormat.h.md) · [`DetailModel.h`](DetailModel.h.md) · [`BufferUtils.h`](BufferUtils.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`r_constants.h`](r_constants.h.md) · [`xrCDB/xrCDB.h`](../../xrCDB/xrCDB.h.md)
**Used by** — [`DetailManager.cpp`](DetailManager.cpp.md) · [`DetailManager_CACHE.cpp`](DetailManager_CACHE.cpp.md) · [`DetailManager_Decompress.cpp`](DetailManager_Decompress.cpp.md) · [`DetailManager_VS.cpp`](DetailManager_VS.cpp.md) · [`DetailManager_soft.cpp`](DetailManager_soft.cpp.md) · [`xrRender_console.cpp`](xrRender_console.cpp.md) · [`dx11DetailManager_VS.cpp`](../xrRenderDX11/dx11DetailManager_VS.cpp.md) · [`glDetailManager_VS.cpp`](../xrRenderGL/glDetailManager_VS.cpp.md)
**Tier floor** — T1: it declares device buffers, a pool allocator over a fixed-capacity arena, and bit-packed per-cell state.

## Purpose

Declares the surface implemented across five files:
[`DetailManager.cpp`](DetailManager.cpp.md) (lifetime, per-frame visibility, dispatch),
[`DetailManager_CACHE.cpp`](DetailManager_CACHE.cpp.md) (the sliding window),
[`DetailManager_Decompress.cpp`](DetailManager_Decompress.cpp.md) (cell → plants),
[`DetailManager_soft.cpp`](DetailManager_soft.cpp.md) and
[`DetailManager_VS.cpp`](DetailManager_VS.cpp.md) (the two draw paths).

## The sizing constants, and what they mean

```text
dm_slot_size       = 2.0     # metres per cell; fixed by the file format
dm_obj_in_slot     = 4       # models per cell; fixed by the file format
dm_max_objects     = 64      # distinct models per level; fixed by the file format
dm_cache1_count    = 4       # cells per side of a coarse visibility block
dm_max_decompress  = 7       # cells decompressed per frame at most (14 in the tools)

dm_size            # cells from the camera to the edge of the cache, derived from the
                   #   detail-radius setting: floor(radius / 4) * 2
dm_cache_line      = 2 * dm_size + 1     # the cache is a square centred on the camera's cell
dm_cache_size      = dm_cache_line squared
dm_cache1_line     = 2 * dm_size / dm_cache1_count
dm_fade            = 2 * dm_size - 0.5   # metres at which an instance has faded to nothing
```

**Invariants**

- `2 * dm_size` **must** be divisible by the coarse block size, because the coarse grid tiles the fine grid exactly. The derivation of `dm_size` from the radius setting — divide by four, floor, multiply by two — exists solely to guarantee that divisibility for any setting the player picks. That is the whole reason for the odd arithmetic.
- The cache line is *odd*: the camera's cell is the exact centre, with the same number of cells in each direction. Everything in the cache-shifting code depends on it.
- The decompression budget of seven cells per frame is the single most important tuning number in the layer. Decompressing one cell costs a collision query against the level's static geometry plus a few thousand ray casts, and the budget is what keeps that off the frame's critical path. The tools raise it to fourteen because they do not have a frame budget.
- All of the above are *runtime* values recomputed when the player changes the detail radius, which is why they are mutable globals rather than constants. Changing the radius rebuilds the manager from scratch; it cannot be changed in place.

## The records

```text
RECORD SlotItem            # one plant
  scale             : real       # authored range, randomized at decompression
  scale_calculated  : real       # scale * distance fade, recomputed per frame
  rotation          : matrix     # yaw-only rotation with the position folded into it
  vis_id            : int        # which of three draw lists: still, wave A, wave B
  c_hemi, c_sun     : real       # baked lighting, from the cell
  c_rgb             : vector3    # only on the oldest renderer, which lights per vertex

RECORD SlotPart           # all instances of ONE model in one cell
  model_id : int
  items    : list<SlotItem>          # every instance, regardless of visibility
  r_items  : list<SlotItem> [3]      # this frame's visible instances, split by vis_id

RECORD Slot               # one decompressed cell
  empty  : bool
  state  : still-pending-decompression | ready
  frame  : int (30-bit)    # when this cell's per-instance work may next be redone
  sx, sz : int             # the cell's world grid coordinates
  vis    : visibility record (box and sphere)
  parts  : list<SlotPart> of length 4

RECORD CacheBlock         # one coarse block: 4x4 cells, tested before they are
  empty  : bool
  vis    : visibility record covering all sixteen
  cells  : list<reference to Slot> of length 16
```

**Invariants**

- `frame`, `empty` and `state` share one machine word, and `frame` gets 30 bits. That is not a space optimization for its own sake: there are up to a quarter of a million `Slot` records and they are swept linearly every frame, so the record's size is its cost. 30 bits of frame counter wraps after about eight months of continuous play at 60 Hz, which is not a concern.
- The three `r_items` lists exist so that instances can be drawn in three batches — non-waving, and two wind phases — without re-sorting. The split is decided once, at decompression, and never changes for the instance's life.
- A `Slot` points into a preallocated pool of exactly `dm_cache_size` records. Cells are never allocated or freed; the grid *rotates* pointers as the camera moves (see [`DetailManager_CACHE.cpp`](DetailManager_CACHE.cpp.md)). `SlotItem`s come from their own fixed-block pool.

## Wind

```text
RECORD SwingValues        # two authored sets: "normal" and "fast"
  rot1, rot2   : real     # rotation amplitudes for the two wave phases
  amp1, amp2   : real     # displacement amplitudes
  speed        : real
```

**Invariants** — The current wind is a linear interpolation between the two authored sets, driven by the weather system's single wind-strength factor. There is no third set and no per-level authoring: wind strength in the shipped data is one number on a curve. The two sets come from the `details` section of the system configuration.

## Exported units

- **`load()` / `unload()`** — open the level's detail file, build the model table, initialize the cache, read the wind parameters; and the reverse.
- **`dispatch_background_calculation()`** — queue this frame's cache maintenance and visibility onto a worker; see [`DetailManager.cpp`](DetailManager.cpp.md).
- **`render(command_list)`** — wait for that work and draw.
- **`cache_*`** — the sliding window; see [`DetailManager_CACHE.cpp`](DetailManager_CACHE.cpp.md).
- **`hw_*` / `soft_*`** — the two draw paths and their resources.
- **`use_vertex_programs()`** — which path this machine gets.
- **`query_cell(x, z)`** — the bounds-checked read of the file's cell array.
