# src/Layers/xrRender/DetailManager_CACHE.cpp

> The sliding window: a square grid of decompressed cells centred on the camera, rotated a row at a time as the camera moves, refilled at a fixed budget per frame, nearest first.

**Needs** — [`DetailManager.h`](DetailManager.h.md) · [`DetailFormat.h`](DetailFormat.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: the grid is an array of pointers into a preallocated arena and the rotation is a pointer shuffle; nothing is allocated or freed after initialization.

## Purpose

The grass layer can only afford to hold a few hundred metres of decompressed plants around the camera. This file is the policy that keeps the right cells decompressed: which cell is where in the window, what happens when the camera crosses a cell boundary, and which pending cells get this frame's decompression budget.

## The window

```text
The cache is a square of (2 * dm_size + 1) cells per side, centred on the
camera's cell. Every cell in it is a pointer into a fixed pool of exactly that
many records. Cells are NEVER allocated, freed, or moved — only the pointers
rotate.

Cache grid coordinates run:
  column x  ->  world cell  (centre_x - dm_size + x)
  row    z  ->  world cell  (centre_z - dm_size + (line - 1 - z))
```

**Invariants**

- **The z axis is flipped between the cache grid and the world.** Row zero of the cache is the *highest* world z. There is no stated reason and no behavioural consequence — the grid is symmetric and every access goes through the conversion helpers — but the asymmetry is real and a rebuild that silently un-flips it while keeping the shift directions will move the whole window one row off. The self-check below exists precisely because this is easy to get wrong.
- A validation pass walks the whole grid and confirms every cell's recorded world coordinates match its grid position. It runs in checked builds only, after initialization and (commented out in the shipped source) after each shift. That it was commented out says the shift was the suspect.
- Every cell in the window is either ready or pending; there is no "absent". A cell outside the level's grid is a *ready, empty* cell, not a missing one.

## `cache_initialize()`

**Contract** — points every grid position at its pool record, queues every one of them for decompression, and builds the coarse blocks' cell reference lists. Called once per level load.

**Invariants** — The coarse block at (bz, bx) covers the sixteen fine cells at (bz·4 + z, bx·4 + x). Those references are set up **once** and never updated, which is only correct because the rotation moves *pointers within* the grid rather than moving the grid: a coarse block always refers to the same sixteen grid positions, whatever cells currently sit in them. This is the reason the rotation is a pointer shuffle and not an index remap.

## `cache_update(camera_cell_x, camera_cell_z, eye)`

**Contract** — brings the window to the camera's cell and spends this frame's decompression budget. Runs on the background task. Does not draw, does not allocate.

```text
FUNCTION cache_update(vx, vz, eye)
  moved = (centre differs from (vx, vz))

  WHILE centre_x /= vx
    shift the whole grid one column toward the camera;
    the column that fell off the far edge becomes the new near column,
    and is re-tasked for the world cell it now represents
  WHILE centre_z /= vz
    the same, by row

  IF every cell in the window is pending          # first frame after a load
    decompress all of them, now                   # a load stall, deliberately
  ELSE
    pick the dm_max_decompress pending cells nearest the eye and decompress those

  IF moved
    rebuild each coarse block's merged bounds and empty flag
```

**Invariants**

- **The shift is a `while`, not an `if`.** A camera that teleports — a level transition, a debug jump — walks the grid one row at a time until it arrives, re-tasking a whole row per step. For a long teleport this is quadratic and slow, and it is correct. The alternative, detecting a jump and re-tasking everything, is what the first-frame branch below effectively does; the code simply does not detect the case.
- **The first frame decompresses the whole window synchronously.** The condition is "every cell is pending", which is only true immediately after initialization. This is a multi-second stall and it is the right call: appearing on a level with no grass and watching it grow in over ten seconds is worse than a longer load.
- **After that, the budget is seven cells per frame, chosen by distance to the eye.** The selection is a bounded nearest-N scan: keep seven best-so-far distances, replace the current worst whenever a closer candidate appears, and re-find the worst. At seven candidates the linear re-scan beats a heap and allocates nothing.
- The chosen cells are removed from the pending list **in descending index order**, because removal is by index and removing a low index invalidates the higher ones. This is a consequence of the list's removal being a swap-with-last, and it is the kind of detail that survives as "remove them in an order that does not invalidate the remaining indices" rather than as itself.
- The coarse blocks' bounds are rebuilt **only when the camera changed cells**, not when a cell was decompressed. A cell that is decompressed without a camera move tightens its own bounds (the decompressor does) but its block's merged bounds stay stale until the next move. The stale bounds are always *larger* than the truth — they came from the cell's pre-decompression box, which covers the file's recorded height range — so the error is conservative and only costs a few redundant fine-grained tests.

## `cache_task(grid_x, grid_z, cell)`

**Contract** — points one grid position at a new world cell: reads the file's record for it, sets the empty flag and the four model ids, releases the plants the cell previously held back to their pool, builds a provisional bounding box from the file's height range, and queues the cell for decompression if it is not already queued.

**Invariants**

- The provisional box is the cell's 2-metre footprint by the file's recorded base and height, grown by a small epsilon. It is used for culling until the decompressor replaces it with the real bounds of the plants it generated. It must be conservative — it must contain everything the decompressor will produce — or cells will be culled before their grass appears.
- A cell already in the pending list is not queued twice. The check is on the cell's own state, not a search of the list, which is what makes re-tasking a whole row cheap.
- An empty cell is still tasked and still decompressed; the decompressor returns immediately. Skipping empty cells here would complicate the pending-list bookkeeping for no gain, since the decompression of an empty cell costs nothing.

## `query_cell(x, z)`

**Contract** — reads the file's record for a world cell, or returns a shared synthetic empty record when the coordinates fall outside the level's grid. Never fails.

**Notes** — The synthetic empty record is a single shared mutable instance whose four model ids are rewritten to the sentinel on every out-of-bounds query. Rewriting it each time is redundant — nothing ever writes anything else into it — and it makes the function non-reentrant for no reason. A rebuild uses an immutable constant. The reason it is a *reference* rather than a copy is that the in-bounds path returns a reference into the mapped file, and the two arms must agree.
