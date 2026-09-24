# src/Layers/xrRender/DetailManager.cpp

> The grass layer's frame: pick this frame's visible plants off a background thread, fade them by distance, and hand three batched lists to a draw path.

**Needs** — [`DetailManager.h`](DetailManager.h.md) · [`HOM.h`](HOM.h.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [`xrCore/Threading/TaskManager.hpp`](../../xrCore/Threading/TaskManager.hpp.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: the visibility sweep walks a quarter of a million cache records per frame with explicit prefetching and packed state, and it runs concurrently with the rest of the frame.

## Purpose

Holds the grass layer's lifetime and its per-frame decision: of the plants the cache currently holds, which are visible, how big are they at this distance, and which of the three wind batches does each belong to. It also holds the dither matrix the decompressor needs and the wind interpolation, both because they have nowhere better to live.

## The concurrency shape — the load-bearing decision

The cache maintenance and the visibility sweep run on a **worker task dispatched early in the frame**, and the draw call waits for it. Everything in the sweep reads only the camera and the immutable level data, and writes only into the cache's own records. Nothing else in the frame touches the cache.

```text
FUNCTION dispatch_background_calculation()
  SPAWN
    IF the grass layer is disabled or the level has no detail file, RETURN
    eye = camera position
    cell_x = floor(eye.x / 2.0 + 0.5)        # the half-cell bias; see DetailFormat.h
    cell_z = floor(eye.z / 2.0 + 0.5)
    cache_update(cell_x, cell_z, eye)        # slide the window, decompress up to 7 cells
    update_visible()                         # the sweep below

FUNCTION render(command_list)
  AWAIT the spawned task
  ... draw ...
```

**Invariants** — The task re-checks that the manager still exists and that the level is still loaded, because a level unload can race the task it dispatched. This is the one place in the layer where the lifetime is not structurally guaranteed, and a rebuild needs some equivalent — a cancellation token, a generation counter — rather than a null test.

## `update_visible()`

**Contract** — clears and refills the three per-model visible lists from the cache. Reads the camera frustum and the occlusion map. Writes only into cache records. Allocates only by growing the visible lists, which reach a steady size after a few frames.

```text
FUNCTION update_visible()
  clear all three visible lists
  frustum = the camera's frustum, left/right/top/bottom and far     # NO near plane
  fade_limit = dm_fade squared ; fade_start = 1.0 squared
  cheap_threshold = 16 * screen_area_discard_threshold

  FOR EACH coarse block IN the 4x4-cell grid
    IF block is empty                          CONTINUE
    result = frustum.test(block.sphere, block.box)      # returns none / partial / full
    IF result = none                           CONTINUE

    FOR EACH cell IN the block's sixteen
      IF cell is empty                         CONTINUE
      IF result = partial AND frustum.test(cell) = none   CONTINUE
      IF NOT occlusion_map.visible(cell)       CONTINUE

      IF current_frame > cell.frame            # the per-cell refresh, see below
        distance_sq = eye distance to cell centre, squared
        IF distance_sq > fade_limit            CONTINUE
        alpha = 0 when inside fade_start, else (distance_sq - fade_start) / fade_range
        cell.frame = current_frame + random integer in [15, 30]

        FOR EACH part IN cell.parts with a model
          clear its three per-frame lists
          model_radius_sq_over_dist = model.radius^2 / distance_sq
          FOR EACH instance IN part.items
            instance.scale_calculated = instance.scale * (1 - alpha)
            screen_area = scale_calculated^2 * model_radius_sq_over_dist
            IF screen_area < discard_threshold    CONTINUE     # too small to see
            list = 0 when screen_area <= cheap_threshold, else instance.vis_id
            append instance to part's list[list]

      FOR EACH part IN cell.parts with a model
        append each non-empty per-frame list to visible[list_index][part.model_id]
```

**Invariants**

- **The frustum has no near plane.** Grass is on the ground under the camera; clipping it against a near plane would cull the plants the player is standing in.
- **The per-cell instance pass is amortized, not per frame.** After a cell's instances are sized, the cell is stamped with a frame number 15 to 30 frames in the future and skipped until then. The *randomized* interval is the load-bearing part: a fixed interval would make every cell in the cache come due on the same frame and produce a periodic hitch. Spreading the due dates flattens the cost. The consequence a rebuild must accept is that an instance's distance fade lags the camera by up to half a second — invisible in motion, and the reason the fade band is metres wide rather than centimetres.
- **The list collection, by contrast, runs every frame**, using whatever sizing the last refresh produced. Visibility must be exact; size need not be.
- **Screen-area culling, not distance culling, decides an instance is too small.** The estimate is the model's radius scaled by the instance and divided by the squared distance — a screen-area proxy — compared against the same global threshold the rest of the renderer uses for discarding small objects. That shared threshold is why turning down detail quality thins grass and distant props together.
- **Wind is a luxury for near plants only.** An instance whose screen area is below sixteen times the discard threshold is forced into the still batch regardless of its authored wind class. Sixteen is a pure magic number with no derivation in the source; it places the wind cutoff at four times the discard *distance*, which is a plausible hand-tuned choice and nothing more.
- The two-level grid exists to make the common case cheap: on a typical frame most of the cache is behind the camera, and the coarse test rejects sixteen cells at a time. When a block tests *fully* inside the frustum, its cells skip their own test entirely — that is the `partial` check.

## `render(command_list)`

**Contract** — waits for the background task, sets the wind for this frame, disables back-face culling for the whole layer, and delegates to one of the two draw paths. Restores the cull mode after.

**Invariants**

- Grass is drawn **double-sided**. Every plant is a handful of crossed quads and half of them face away; culling them would halve the foliage. This is set once around the whole layer rather than in the material, which is a fixed-function-era habit — a rebuild puts it in the material and deletes the bracket.
- A global shader flag is raised for the duration of the layer and lowered after. Materials sample it to branch on "am I grass", which is how a shared material template serves both grass and ordinary geometry. It is a global mutable register in the middle of a draw path and it is exactly the kind of thing a rebuild replaces with a material variant.
- The wind is interpolated from the two authored sets by the weather system's wind-strength factor, once per frame, not per plant.

## `use_vertex_programs()`

**Contract** — reports whether this machine gets the hardware path. True when the device has any programmable vertex stage and the renderer is not forced into fixed-function mode.

**Notes** — On any machine built this century the answer is always true and the software path is dead code. It survives because the oldest supported renderer generation can be forced into fixed-function operation by a console setting. A rebuild targeting modern hardware deletes [`DetailManager_soft.cpp`](DetailManager_soft.cpp.md) entirely.

## The dither matrix

**Contract** — builds a 16×16 ordered-dither threshold matrix, once, at level load, from a 4×4 seed pattern.

```text
FUNCTION build_dither_map(levels, out 16x16)
  step = 255 / (levels - 1)
  factor = (step - 1) / 16
  FOR i, j, k, l EACH IN 0..3
    out[4k + i][4l + j] = round(seed[i][j] * factor + (seed[k][l] / 16) * factor)
```

**Invariants** — The seed is the classic 4×4 ordered-dither (Bayer) matrix. Expanding it to 16×16 by *nesting a scaled copy of itself* is the standard recursive construction, and the reason given in the original is the honest one: a bare 4×4 leaves a visible repeating pattern in the grass and gives only seventeen distinguishable density levels. The expansion gives 256.

The matrix is built for two levels — a pure black-and-white threshold — because the decompressor uses it as a yes/no test: does a plant exist at this grid point. It is not being used to dither an image. That is the whole trick of the grass layer's placement: an ordered dither of a continuous density field gives a deterministic, evenly-spread, non-periodic set of points, for the cost of one table lookup and one comparison per candidate.
