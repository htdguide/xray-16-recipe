# src/xrGame/DynamicHeightMap.cpp

> A scrolling cache of ground height around the camera, rebuilt a slot at a time by raycasting straight down onto the level's static collision geometry. Dead code in this branch.

**Needs** — [`DynamicHeightMap.h`](DynamicHeightMap.h.md) · [`xrCDB/Intersect.hpp`](../xrCDB/Intersect.hpp.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`DynamicHeightMap.h`](DynamicHeightMap.h.md)
**Tier floor** — T2: a fixed grid of floats, amortized ray casts against the collision tree

## Purpose

The problem: something wants to know the ground height at an arbitrary horizontal point,
cheaply, many times per frame, over a region that follows the camera. Raycasting per query
is too expensive; precomputing the whole level is too much memory. The answer here is a
small fixed grid of cached height samples that *scrolls* with the camera and refills its
newly exposed edge over subsequent frames.

Nothing in the shipped engine calls it. It is recorded here because the technique is
reusable and because its parameters encode a real judgement about cost.

**The geometry of the cache.** A square of nine-by-nine slots centred on the camera's slot,
each slot four metres on a side and holding a sixteen-by-sixteen grid of height samples.
So the cache covers thirty-six metres square at a quarter-metre resolution, in about
twenty thousand floats. The odd slot count is what makes "centred" meaningful; the four
slots of margin in each direction are what give the refill time to happen before the
player reaches the edge.

**The amortization.** Moving one slot's width exposes nine slots of new territory at once,
but only one slot is rebuilt per frame. That is the cost ceiling: whatever happens, this
system casts at most two hundred and fifty-six rays per frame. The consequence is that
newly exposed ground is stale for up to nine frames, and a camera moving faster than one
slot per nine frames — four metres in nine frames, an ordinary sprint — outruns the
refill permanently.

## State

```text
RECORD Slot
  samples : 16 x 16 grid of real   # ground height; "no ground" is the minimum float
  ready   : bool                   # false while queued for rebuild
  x, z    : int                    # which slot of the world grid this holds

RECORD StaticHeightMap
  pool    : 81 slots               # the storage; never reallocated
  grid    : 9 x 9 references into the pool    # the scrolling window
  c_x,c_z : int                    # the world slot the window is centred on
  queue   : list of slots awaiting rebuild
  polys   : list of upward-facing triangles   # scratch, reused per rebuild

RECORD HeightMap
  static_map, dynamic_map
  last_frame : int                 # so the per-frame update runs once however many queries
```

Invariants: the window holds references into a fixed pool and scrolling *rotates the
references*, never copies a slot's samples. A slot leaving one edge is the same storage
that reappears at the other, which is why scrolling is cheap and why the reappearing slot
must be re-stamped with its new world coordinates and queued.

## `CHM_Static::Update` — scrolling

**Contract** — compares the camera's slot against the window's centre and, if they differ,
shifts the window by one slot in each differing axis. Each shift recycles the row or column
leaving the window to the opposite edge, marks it not-ready and queues it. Only one step
per axis per call — the window cannot jump.

```text
FUNCTION scroll()
  v_x, v_z = the camera's slot coordinates
  IF v_x > c_x THEN shift the window one slot in +x, recycling column 0 to the far edge
  ELSE IF v_x < c_x THEN shift one slot in -x, recycling the far column to column 0
  (the same for z)
  # each recycled slot: mark not ready, queue it, stamp its new world coordinates
```

**Notes** — because the window moves one slot per call and a call happens once per frame, a
camera teleport leaves the window crawling toward the new position for as many frames as
there are slots between. Nothing detects the teleport.

## `CHM_Static::Update` — rebuilding a slot

**Contract** — pops one queued slot per frame and fills its samples. Selects the static
triangles overlapping the slot's column of space, discards downward-facing ones, then for
each sample position casts a ray straight down and keeps the *highest* intersection. A
slot with no triangles at all is cleared to "no ground".

```text
FUNCTION rebuild(slot)
  volume = the slot's four-metre square, extended from 20 m below the camera
           to 100 m above it
  triangles = static collision query against volume
  IF none THEN slot.clear(); RETURN

  polys = those triangles whose normal points upward     # floors, not ceilings
  FOR EACH sample position (x, z) in the slot's 16 x 16 grid
    cast a ray downward from the top of the volume
    height = the highest hit among polys, or (volume bottom - 5) if none
    slot.samples[z][x] = height
```

**Invariants** — the vertical extent is *relative to the camera*, not absolute: a hundred
metres up and twenty down. So the cache only ever sees ground within that band, and a slot
rebuilt while the camera is on a rooftop records the rooftop, not the street. This is what
makes the structure a "height map near the camera" rather than a terrain height map.

**Notes**

- Keeping the highest hit rather than the first means the cache records the topmost walkable
  surface — a bridge rather than the ground beneath it — which is the right answer for
  anything following the camera's own level.
- The "no ground" value is five metres below the sampled band rather than the float
  minimum used by `clear`, so two different sentinels mean the same thing. A rebuild
  should pick one.
- The scratch triangle list is filled per slot and **never cleared between slots**, so
  each rebuild tests against every triangle accumulated since the map was created. This is
  a defect, and it is the reason the amortization budget above is not actually a ceiling.

## `CHM_Static::Query`

**Contract** — reads the cached height at a horizontal position: locate the slot, clamp
into the window, locate the sample within the slot, clamp again, return it. Never casts
anything. A position outside the window silently returns the nearest edge's value rather
than failing.

**Notes** — the slot index is computed as the position's slot minus the centre minus the
margin, which is off by the margin in the wrong direction: the correct expression adds the
margin. As written, every query lands in the lower-left quadrant of the window or is
clamped to its edge. Combined with the uncleared scratch list above, the conclusion is
that this code never ran in anger.

## `CHeightMap::Query`

**Contract** — the public entry point. Refreshes both layers at most once per frame however
many times it is called, then answers with the greater of the two layers' heights. Taking
the maximum is how a dynamic surface — a lift platform, a vehicle roof — would override the
static ground beneath it.

## `CHM_Dynamic`

**Contract** — the intended second layer, for surfaces that move. Entirely unimplemented:
its update does nothing and its query always answers "no ground", which the maximum above
discards. A rebuild wanting this structure must write it.

## Could not recover

- The three-dimensional ray query declared on the public height map has no implementation
  anywhere. What it was to return — a surface point along a ray — is legible from its
  signature and nothing else.
- Why the cache is four metres per slot at sixteen subdivisions rather than any other
  split of the same resolution is not recorded; the product is what matters and the split
  only affects the refill granularity.
- No caller exists for any of this, so the intended consumer — grass placement, a minimap,
  creature foot placement are all plausible — cannot be recovered from the tree.
