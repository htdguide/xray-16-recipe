# src/Layers/xrRender/r__occlusion.cpp

> A pool of hardware occlusion queries handed out and returned in a strict pattern, so that the smallest possible set of query objects is reused frame after frame.

**Needs** — [`r__occlusion.h`](r__occlusion.h.md) · [`QueryHelper.h`](QueryHelper.h.md) · [`R_Backend.h`](R_Backend.h.md) · [`xrRender_console.h`](xrRender_console.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`r__occlusion.h`](r__occlusion.h.md)
**Tier floor** — T1: it owns device query objects with explicit creation and release, spins on a device result with a timeout, and is locked because several render contexts issue queries concurrently.

## Purpose

Occlusion queries are the renderer's hardware visibility test: draw a bounding volume, ask how many fragments survived. Every light ([`light_vis.cpp`](light_vis.cpp.md)), every shadow-caster candidate ([`light_smapvis.cpp`](light_smapvis.cpp.md)) and the pixel-coverage tool ([`r__pixel_calculator.cpp`](r__pixel_calculator.cpp.md)) uses them.

Query objects are a scarce device resource and creating one mid-frame is expensive, so they are pooled. The pool's whole design rests on one observation about how they are used, and the file states it as a requirement on callers:

> Allocations and frees come in matched rounds. Every query allocated in a round is freed before the next round begins, and within a round the order is stable. Therefore: use as few distinct queries as possible, and always prefer the one allocated earliest.

That is what makes the pool converge on a working set roughly the size of one frame's query count, no matter how many were ever created.

## State

```text
RECORD QueryPool
  enabled  : bool                  # a command-line switch disables the whole mechanism
  free     : list<Query>           # sorted by ORDER, descending; taken from the back
  in_use   : list<Query>           # indexed by the handle given out
  free_ids : list<int>             # handles returned to be reused
  lock     : mutex

RECORD Query
  device_query : device object
  order        : int      # the query's rank — its position in the "prefer earliest" order
```

Invariants:

- `free` is kept sorted by `order` **descending**, so the *lowest*-ordered query is at the back and a pop from the back is the cheapest possible "prefer the earliest". A return inserts at the position that maintains the ordering, by a linear scan from the back — correct because a returned query is almost always near the end.
- A handle is an index into `in_use`. `free_ids` recycles handles so `in_use` does not grow unboundedly; when it is empty, a new handle is the current size.
- Every operation takes the lock. Several render contexts issue and read queries concurrently, and the pool is shared.
- The invalid handle is a reserved sentinel. Every operation accepts it and does nothing, so a caller whose allocation failed need not branch.

## `create`

**Contract** — Pre-create up to a stated number of query objects, stopping early if the device refuses. Reads the command line for the disable switch. Called at device bring-up.

After creation the list is reversed, because the objects were made in increasing order and the pool wants them in decreasing order.

**Notes** — The initial size is a per-context base count times the number of parallel contexts, doubled. It is a provisioning guess, not a limit: the pool grows on demand. Its only purpose is to avoid a burst of mid-frame creations on the first few frames.

## `destroy`

**Contract** — Release every query object, in use or free, and empty all three lists. Called at device teardown. A query still outstanding at this point loses its result, which is correct — the frame is being abandoned.

## `begin`

**Contract** — Take a query from the pool, start it on the device, and return both a handle (by reference) and the query's *order*. The order is returned because a caller may need to know the relative age of its query — the light visibility path records it for the deferred path's sorting.

```text
FUNCTION begin(OUT handle) -> order
  IF disabled THEN handle stays as it is ; RETURN 0
  LOCK pool DURING
    IF free is empty
      # Grow. The new query's order is the current in-use count, which places
      # it after every query currently outstanding.
      new = create_device_query()
      IF creation failed
        warn, at most once every forty frames    # this is a flood, not an event
        handle = invalid ; RETURN 0
      IF in_use is at capacity
        reserve another base count in both lists
      insert new at the FRONT of free   # highest order, taken last

    IF free_ids is not empty
      handle = free_ids.pop()
      in_use[handle] = free.pop_back()
    ELSE
      handle = in_use.count
      in_use.append(free.pop_back())

    begin_device_query(in_use[handle])
    RETURN in_use[handle].order
```

**Invariants** — Growth reserves capacity in *both* lists together, because a handle indexes one and the pool holds the other; letting either reallocate independently would invalidate references held across the lock.

## `end`

**Contract** — Stop the query on the device. Does not release it; the result is not available yet. Accepts the invalid handle.

## `get`

**Contract** — Wait for a query's result, return the fragment count, and release the query back to the pool. Blocks. Accepts the invalid handle, for which it returns the "fully visible" sentinel so a caller with no query behaves as if everything is visible.

```text
FUNCTION get(INOUT handle) -> fragments
  IF disabled OR handle is invalid THEN RETURN all_ones      # "visible"
  LOCK pool DURING
    start a timer
    WHILE the device says the result is not ready
      yield to another thread; if there was none to yield to, sleep for the
        configured interval
      IF the timer passes half a second
        fragments = all_ones      # give up and assume visible
        BREAK

    IF fragments = 0 THEN count an occlusion-cull in the frame statistics

    insert the query back into `free` at the position that keeps the list
      sorted by order descending
    release the handle into free_ids
    handle = 0
    RETURN fragments
```

**Invariants**

- The timeout resolves to *visible*, never to hidden. A device that has stopped answering must not make the world disappear. Half a second is an authored constant; at sixty frames per second it is thirty frames, which is far past any legitimate stall, so reaching it means something is wrong and the safe answer is the only consideration.
- The wait yields the processor rather than spinning, and sleeps only when there was nothing to yield to. The sleep interval is configurable and defaults to zero — a bare yield — because on a loaded machine any real sleep costs more frame time than the wait saves.
- The handle is zeroed on return, not invalidated. That is a wart: a caller that reads its handle after `get` sees a valid-looking zero. Callers are expected not to.

**Notes** — Holding the lock across the *wait* serialises every context's query reads. That is not an oversight — the device's query results are read through a shared immediate context on the backend this was written for — but it is the mechanism's main scalability limit, and a rebuild with per-context query readback should take the lock only around the pool bookkeeping.
