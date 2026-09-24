# src/Layers/xrRender/R_DStreams.cpp

> The two per-frame scratch buffers every piece of generated geometry is written into: a vertex ring and an index ring, both append-until-full then wrap-and-discard.

**Needs** — [`R_DStreams.h`](R_DStreams.h.md) · [`BufferUtils.h`](BufferUtils.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`R_DStreams.h`](R_DStreams.h.md)
**Tier floor** — T1: the whole module is a byte-offset allocator over a mapped device buffer, and its correctness rests on the device's guarantee that appending to a region the graphics processor has already read past does not stall.

## Purpose

Most geometry in the engine is loaded once and lives in immutable device buffers. A significant minority is generated every frame and thrown away: particle quads, decals, the grass layer's instances, sky and weather geometry, screen-space quads, debug lines, every immediate-mode thing. Allocating a buffer per batch would be ruinous; so would writing into a buffer the graphics processor is still reading.

These two objects solve that with one shared ring per resource kind. A caller asks for room for *n* vertices (or indices), gets a writable pointer and the offset it was placed at, fills it, and says how much it actually used. The ring gives out space linearly until it cannot fit the request, then wraps to the start and tells the device to **discard** the old contents — which is the device's promise that a fresh block of memory will be handed over and no synchronisation is needed. Between discards, every map is an **append**: the caller may write only into the newly reserved region and may not touch anything already given out.

There is exactly one vertex stream and one index stream in the renderer, shared by everything.

## State

```text
RECORD Stream                     # the vertex and index rings differ only in element size
  device_buffer : BufferHandle    # dynamic, write-only, mapped per allocation
  size          : int (bytes)     # both rings measure their capacity in bytes
  position      : int             # the write cursor — BYTES in the vertex ring,
                                  # INDEX COUNT in the index ring. See the note below.
  discard_id    : int             # incremented on every wrap; see below
  previous_buffer : BufferHandle  # kept across a device reset, see reset_begin
```

**Invariants**

- `position <= size` always; a request that cannot fit wraps rather than failing.
- **A request larger than the whole ring is a hard error, not a wrap.** The check is made before anything else, because wrapping would silently produce a buffer that cannot hold the data and the resulting corruption would appear far away.
- The request count must be non-zero. A zero-vertex batch is a caller bug: it would map a zero-byte region and hand back a pointer nothing may be written through.
- Exactly one map may be outstanding at a time. There is no locking and no free list; the second concurrent map would hand out the same region. The debug build asserts this explicitly, which is the only reason a rebuild can know it is a rule rather than an accident.
- **`discard_id` is the cache key for everything built out of the ring.** A generator that records `(offset, count, discard_id)` alongside its output can ask later whether that output is still in the buffer: if the counter has not moved and the count is unchanged, the same offset still holds the same data and the generation can be skipped entirely. The skinning path is the reason this exists — a skinned mesh drawn more than once in a frame (a depth pre-pass, then shading, then a shadow map) is transformed once and drawn three times from the same region. Once the ring wraps, every recorded offset is stale. The counter must be monotonic rather than a wrap flag, because "has it wrapped since I looked" is the actual question.

## Sizes

```text
vertex ring : 4096 KiB    # ~4 MB
index ring  :  512 KiB    # 256 K indices at two bytes each
```

**Notes** — Both are console variables read once at creation, so a rebuild may expose them the same way. The ratio is the load-bearing part: generated geometry is overwhelmingly quads and particle sprites, whose index-to-vertex byte ratio is far below one. Sizing the index ring equal to the vertex ring would waste memory that the vertex side would rather have; sizing it much smaller makes the index ring wrap far more often than the vertex ring, which invalidates every cached batch on both sides. The two are chosen so they wrap at roughly the same rate for the traffic the engine actually generates.

## `Create` / `Destroy`

**Contract** — `Create` first asks the resource manager to **evict** cached resources, then allocates the ring at its configured size, zeroes the cursor and the discard counter, and logs the size. `Destroy` releases the buffer and zeroes everything.

**Notes** — Evicting before allocating is not politeness: this is one of the few large, contiguous, device-visible allocations the engine makes, and it is made at a moment (startup, or a device reset) when the resource manager may be holding a great deal of memory it no longer needs. Failing this allocation is fatal; failing an eviction is not.

## `Lock` — reserve room in the vertex ring

**Contract** — Given a vertex count and a byte stride, returns a writable pointer to room for that many vertices and, separately, the **vertex index** at which they were placed (not a byte offset — the caller passes it to the draw call as a base vertex). Fails hard on a zero count or a request larger than the ring. Blocks only insofar as the device's map does.

```text
FUNCTION lock_vertices(count, stride) -> (pointer, first_vertex)
  bytes_needed = count * stride
  FAIL WITH "request exceeds ring" IF bytes_needed > size OR count == 0

  capacity_in_vertices = size / stride
  next_vertex          = position / stride + 1     # note the +1

  IF count + next_vertex >= capacity_in_vertices
    # wrap: the ring cannot fit this at the cursor
    position     = 0
    first_vertex = 0
    discard_id   = discard_id + 1
    map_mode     = discard            # device hands back fresh memory
  ELSE
    position     = next_vertex * stride
    first_vertex = next_vertex
    map_mode     = append             # device must not move what is already there

  pointer = map(device_buffer, at = position, length = bytes_needed, mode = map_mode)
  RETURN (pointer, first_vertex)
```

**Invariants**

- The cursor is converted to a vertex index and **rounded up by one** before use. The ring is shared by callers with different strides, so the byte cursor left by the previous caller is almost never a multiple of this caller's stride; rounding up is what re-aligns the cursor to a whole vertex of the *current* stride. The cost is up to one stride of wasted space per allocation, which is the price of one ring serving every vertex format instead of one ring per format.
- The wrap test uses `>=` against the capacity, not `>`. Combined with the rounding, this leaves at least one vertex slot unused at the end of the ring. It is a deliberate margin, not an off-by-one to be tightened: it guarantees that the rounded-up cursor of the *next* allocation is still inside the buffer.

## `Unlock` — commit the vertex reservation

**Contract** — Takes the count the caller *actually* wrote and its stride, advances the cursor by that much, and unmaps. The committed count may be less than the reserved count — generators that cannot predict their output (a particle emitter that culls as it writes) reserve an upper bound and commit the truth. It may never be more.

**Notes** — Reserving pessimistically and committing exactly is the pattern the whole dynamic-geometry layer is built on, and it is what makes the "one map at a time" rule tolerable: a generator does not need to know its output size in advance, only a bound on it.

## `Lock` / `Unlock` — the index ring

**Contract** — The same discipline with one simplification: indices are always two bytes, so there is no stride and no re-alignment. Returns a pointer and the index offset to pass as the draw's first index. Wraps on the same `>=` test, bumping the same kind of discard counter.

**Notes** — The index ring's cursor is counted in *indices* while its capacity is counted in *bytes*, so every comparison between them carries an explicit factor of two, and the vertex ring keeps both in bytes. The asymmetry is real and is the single easiest thing in this file to get wrong; a rebuild should pick one unit per ring and say which, rather than reproducing the mixture.

## `Flush`

**Contract** — Forces the next allocation to wrap by parking the cursor at the end. Both rings are flushed at the start of every frame, so the first allocation of a frame always discards and always lands at offset zero. That is what makes the discard counter a usable frame-to-frame cache key: a batch's recorded counter can only still be current if the ring has not wrapped *within* the frame, which is exactly the condition the counter tests.

## `reset_begin` / `reset_end` — surviving a device reset

**Contract** — On a device-lost or resize event, `reset_begin` stashes the old buffer handle and destroys the ring; `reset_end` recreates it. The stashed handle is deliberately **not** cleared afterwards.

**Notes** — Keeping the stale handle is the load-bearing oddity here. Every geometry record the resource manager knows about still names the pre-reset buffer; after the rings are recreated, the manager walks its geometry list and rewrites any record whose buffer equals a stashed handle to the new one (see [`ResourceManager_Reset.cpp`](ResourceManager_Reset.cpp.md)). Clearing the stash on recreation would leave the manager with no way to tell "this geometry pointed at the pre-reset ring" from "this geometry pointed at some other buffer entirely", and every dynamic geometry record would have to be rebuilt from scratch.
