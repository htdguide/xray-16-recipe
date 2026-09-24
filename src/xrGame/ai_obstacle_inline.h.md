# src/xrGame/ai_obstacle_inline.h

> The obstacle's laziness: every reader forces the computation first, and a move is the only thing that invalidates it.

**Needs** — [`ai_obstacle.h`](ai_obstacle.h.md)
**Used by** — [`ai_obstacle.cpp`](ai_obstacle.cpp.md) · [`ai_obstacle.h`](ai_obstacle.h.md)
**Tier floor** — T3: a cached computation guard.

## Purpose

Computing which navigation vertices an object blocks costs a bounding-box fit over every
visible bone plus a scan of a region of the navigation grid. It must not run per frame for
every object. This file holds the policy that keeps it from doing so.

## Construction

**Contract** — Binds the obstacle to a game object, marks the result stale, and seeds the
oriented box with a placeholder: an axis-aligned half-metre-by-one-metre-by-half-metre box
whose centre is lifted one metre. That placeholder is what `min_box` returns, because
`min_box` does *not* force the computation — the box a caller sees is a standing-human
approximation, not the object's real shape, unless something else forced the fit first.
A rebuild should either force the computation there or name the accessor for what it
returns.

## `compute` and the readers

**Contract** — `compute` runs the real fit once and then never again until invalidated;
`area` and `crc` each force it before answering. `danger_area` returns a list that is never
filled — its computation call is commented out, so the wider "dangerous to stand near"
region the type declares does not exist. Callers of it receive an empty set.

```text
FUNCTION compute()
  IF actual THEN RETURN
  actual = true            # set before, not after: the computation does not re-enter,
                           #   and setting it first makes that explicit
  compute_impl()
```

**Notes** — Invalidation is by movement only (`on_move`). An object that changes *pose*
without moving — a stalker crouching, a door swinging on a hinge whose origin is fixed —
keeps a stale blocked set. That is a deliberate trade: pose changes every frame and the
navigation grid cannot be re-scanned at that rate.
