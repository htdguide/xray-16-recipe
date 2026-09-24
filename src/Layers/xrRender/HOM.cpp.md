# src/Layers/xrRender/HOM.cpp

> Rasterize the level's authored occluders into a low-resolution depth hierarchy once per frame, then answer "is this box behind something" in a few comparisons — with a per-triangle and a per-object skip schedule so that neither side costs what it should.

**Needs** — [`HOM.h`](HOM.h.md) · [`occRasterizer.h`](occRasterizer.h.md) · [`xrCDB/xrCDB.h`](../../xrCDB/xrCDB.h.md) · [`xrCore/Threading/ParallelFor.hpp`](../../xrCore/Threading/ParallelFor.hpp.md) · [Seam: Static collision database](../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — reached through its declarations in [`HOM.h`](HOM.h.md); callers name that, not this file.
**Tier floor** — T1: it runs a software rasterizer over a shared buffer on a worker thread while the rest of the frame reads it.

## Purpose

Sectors and portals cull what is in another room. This culls what is *behind a wall in the same room* — the second half of the engine's visibility answer. It is a **hierarchical occlusion map**: a small software depth buffer built from an authored set of large flat occluding surfaces, with a mip pyramid over it so a bounding box can be rejected against a coarse level without touching every pixel.

Both this and hardware occlusion queries are present in the engine; the chapter-4 README notes that a rebuild picks one. This is the one that costs no graphics-device round trip and therefore no latency, which is why it survived.

## The occluder set — a separate authored asset

```text
The level ships a file of occluder polygons: triangles with a flag.
They are NOT the level's geometry. An artist places a few hundred large
quads where the level's real walls are, and those are all the engine
rasterizes.

RECORD OccluderTriangle
  plane      : plane
  centre     : point
  area       : real
  flags      : bit set        # one bit: this occluder is two-sided
  adjacent   : three references to neighbouring triangles, or none
  skip_until : int            # the frame this triangle may next be considered
  raster     : three screen-space points, scratch for the rasterizer
```

**Invariants**

- **Occluders are authored, not derived.** The whole approach depends on there being a few hundred of them rather than a hundred thousand. A level with no occluder file loads and runs with the map disabled — the warning is logged and everything is visible.
- Vertices are merged within a centimetre when the set is built, and **adjacency between triangles is computed at load**. The rasterizer uses adjacency to avoid double-covering shared edges; see [`occRasterizer.cpp`](occRasterizer.cpp.md).
- A triangle's area is computed from its edge lengths (Heron's formula) and a degenerate one is *logged, not rejected*. It stays in the set and rasterizes to nothing. This is an authoring diagnostic rather than a runtime guard.
- The tree over the triangles is built by the collision database and **cached to disk**, keyed by a checksum of the source file. On a second load of the same level the tree is deserialized instead of rebuilt. Both the caching and the checksum check can be disabled from the command line, which is how a corrupt cache is worked around.

## Building one frame's map

**Contract** — runs on a worker task dispatched early in the frame; the visibility pass waits for it. Reads the camera and the immutable occluder set; writes the shared rasterizer buffer. Nothing else may touch that buffer while it runs.

```text
FUNCTION build(frustum)
  clear the rasterizer
  compose the viewport transforms: one into rasterizer pixels, one into the unit square
  triangles = the tree's triangles intersecting the frustum
  drop those whose skip schedule has not expired
  sort the rest by distance from the camera, NEAREST FIRST

  FOR EACH triangle
    next_skip = current frame + a random 3 to 10 frames

    IF it is one-sided AND the camera is behind its plane
      schedule it to skip; CONTINUE               # facing away: occludes nothing

    clip it against the NEAR PLANE ONLY
    IF nothing survives
      schedule it to skip; CONTINUE

    rasterize each triangle of the clipped polygon
    IF it covered no pixels
      schedule it to skip; CONTINUE

  propagate the buffer into its mip hierarchy
```

**Invariants**

- **Nearest first is the whole reason this is cheap.** A near occluder fills the buffer with shallow depths, and every farther occluder then rasterizes into a buffer that already rejects it early. Sorting by distance to the triangle's *centre* is an approximation that is wrong for large triangles seen edge-on and is right often enough.
- **Only the near plane clips.** The side and far planes are not needed: the rasterizer clamps to its own bounds, and a triangle extending past the far plane still occludes. Clipping against the near plane is mandatory because the projection divides by w.
- **The skip schedule is the per-triangle amortization.** An occluder that contributed nothing — facing away, clipped out, or covering no pixels — is not reconsidered for three to ten frames. The interval is randomized for the same reason the grass layer randomizes its refresh: a fixed interval would make every skipped occluder come due together. The cost of the approximation is that an occluder can be stale for up to ten frames after the camera moves such that it *would* matter; at 60 Hz that is a sixth of a second of slightly over-drawing, which is invisible.
- The two viewport transforms differ between the two graphics backends by a **negated y**, because the two APIs disagree on which way the screen's vertical axis runs and the rasterizer works in device pixels.
- The transforms are recomposed **every frame**, with a comment saying so, because the camera changes. That is obvious and the comment exists because it once was not done.

## `visible(visibility record)` — the cached query

**Contract** — tests an object's bounding box against the map, and schedules when it may next be tested. Called for every candidate object in the scene, several thousand times a frame. Thread-safe against concurrent callers only in that each object's record is touched by one caller.

```text
FUNCTION visible(record)
  IF the record's next-test frame is in the future   RETURN visible
  IF the map is disabled or the box is invalid       RETURN visible

  result = test the box against the map

  IF result is visible
    next test in a random 10 to 25 frames         # cheap to be wrong
  ELSE
    next test next frame                          # expensive to be wrong

  remember the frame tested
  RETURN result
```

**Invariants**

- **The asymmetry is the decision.** A *visible* object is re-tested rarely, because being wrong costs one object drawn that did not need to be. A *hidden* object is re-tested every single frame, because being wrong costs an object that should be on screen and is not — a visible artefact. The original's own comment enumerates the three cases; this is the rule that falls out of them.
- An object with no valid bounding box is visible. That covers objects whose bounds have not been computed yet, which is a normal state on the frame an object spawns.
- Ten to twenty-five frames of staleness on a visible object is a sixth to half a second. An object that becomes occluded therefore keeps being drawn for up to half a second. That is the cost, and it is paid to keep the query count down.

## `visible(box)` — the uncached query

**Contract** — projects the box's eight corners into the unit square, takes their screen extent and nearest depth, and tests that rectangle against the map's hierarchy.

```text
FUNCTION visible(box)
  IF the box contains the camera    RETURN visible        # see below
  project corner 0; IF it is at or behind the near plane, RETURN visible
  FOR EACH remaining corner
    project it; IF it is at or behind the near plane, RETURN visible
    extend the screen rectangle; keep the nearest depth
  RETURN rasterizer.test(rectangle, nearest depth)
```

**Invariants**

- **A box containing the camera is always visible.** Its projection is meaningless.
- **Any corner behind the near plane makes the box visible.** This is the conservative answer: a box straddling the near plane cannot be given a screen rectangle at all. The test is `z < epsilon` on the unprojected depth, which catches both behind and exactly on.
- The rectangle is the **screen bounds of the projected corners**, which is larger than the projected box — conservative in the right direction: a box is only culled when its whole bounding rectangle is behind an occluder at its *nearest* depth. Nothing is ever wrongly culled; things are sometimes wrongly kept.
- The first corner is handled by a separate routine that *initializes* the extents rather than extending them. That is a branch removed from an eight-iteration loop, and it is why the same twelve lines appear twice.

## `visible(polygon)` and `visible(screen rectangle, depth)`

**Contract** — the same conservative projection for an arbitrary polygon (used on a portal's clipped outline), and a direct pass-through for callers that already have a screen rectangle and a depth.

## Enable, disable, statistics

**Contract** — the map can be turned off at any time and every query then answers visible. Enabling only takes effect when a level's occluder set was actually loaded. The developer overlay reports the frame's occlusion time, how many occluders were in the frustum, how many actually rasterized, and how many the level has in total.

**Notes** — The ratio of *rasterized* to *in frustum* is the number that tells an artist whether the level's occluders are doing anything. A level where those numbers diverge has occluders placed where they never face the camera.

The debug draw paints the occluder set over the scene as translucent triangles plus wireframe, with the depth test relaxed so it is visible through geometry. It is the only way to see what an occluder file actually contains, and a rebuild should keep the equivalent — occluder placement is an authoring task that cannot be done blind.
