# src/Layers/xrRender_R2/SMAP_Allocator.h

> Packs square shadow rectangles into one square atlas, first fit, and reports when the
> next one will not fit.

**Needs** — [`r2_types.h`](r2_types.h.md)
**Used by** — [`r2.h`](r2.h.md) · [`r2_R_lights.cpp`](r2_R_lights.cpp.md) · [`r2_rendertarget_phase_smap_S.cpp`](r2_rendertarget_phase_smap_S.cpp.md)
**Tier floor** — T3: integer rectangle arithmetic over a small list; no device contact.

## Purpose

All shadow-casting spot lights share one square depth texture. Each wants a square
sub-rectangle whose side comes from its screen importance. This allocator answers "does
another one still fit, and where" for a single pass over the atlas; when it says no, the
caller closes the batch, renders and accumulates everything placed so far, and starts a
fresh atlas with a new batch identifier. The allocator is therefore deliberately
*non-freeing*: nothing is ever released, the whole pool is reset between batches.

A separate file because the packing decision is the only interesting part of light
batching and is worth reading on its own.

## State

```text
RECORD Rect
  min : (int, int)
  max : (int, int)          # inclusive; a rectangle of side s has max = min + (s-1)

RECORD Allocator
  pool_side       : int             # the atlas is pool_side x pool_side
  placed          : list<Rect>      # every rectangle handed out since the last reset
  candidates      : list<(int,int)> # top-left corners worth trying next
```

**Invariants** — no two rectangles in `placed` overlap; every rectangle lies wholly
inside `[0, pool_side)` on both axes; every candidate corner lies inside the pool.
`candidates` is not a free list — it is a set of *positions* that were made reachable by
an earlier placement, and a position may turn out to be unusable for a given size.

## `reset`

**Contract** — sets the pool side and empties both lists. Called once per batch.

## `place`

**Contract** — given a requested square side, either writes back the rectangle it was
given and returns success, or returns failure leaving the allocator untouched. Does not
allocate device memory and does not block. Requires `4 < side <= pool_side`.

```text
FUNCTION place(side) -> optional<Rect>
  IF placed IS EMPTY
    r = square at (0, 0) of this side
    admit(r)
    RETURN r

  FOR EACH corner IN candidates            # first fit, in the order corners were made
    r = square at corner of this side
    IF r extends past the pool on either axis THEN CONTINUE
    IF r overlaps any rectangle in placed  THEN CONTINUE
    remove corner from candidates
    admit(r)
    RETURN r

  RETURN none                              # batch is full

# admit: record the rectangle and offer the two corners it exposes
FUNCTION admit(r)
  append r to placed
  right = (r.max.x + 1, r.min.y)
  below = (r.min.x,     r.max.y + 1)
  IF right is inside the pool THEN append right to candidates
  IF below is inside the pool THEN append below to candidates
```

**Notes** — the candidate set is what keeps this cheap: instead of scanning the atlas, it
only ever tries the corner immediately right of and the corner immediately below each
rectangle already placed. Because the caller sorts its lights by descending size before
each pass, large rectangles are placed while the atlas is empty and the small ones fill
the staircase they leave behind — which is why first fit is good enough and a real
bin-packer would buy almost nothing.

The overlap scan is linear in the number of rectangles placed, and the corner scan is
linear in the corners, so placement is quadratic in the batch size. That is acceptable
only because a batch is a handful of lights; a rebuild that raised the atlas size by an
order of magnitude would want an interval structure here.

The lower bound of four on the requested side is a guard against a degenerate rectangle
whose one-texel border (the shadow lookup insets by one texel on each edge to avoid
bleeding between neighbours) would consume the whole rectangle.
