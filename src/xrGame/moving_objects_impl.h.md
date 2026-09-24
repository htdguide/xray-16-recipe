# src/xrGame/moving_objects_impl.h

> The six numbers that set how far ahead obstacle avoidance looks, how finely it samples, and how stubborn a waiting creature is.

**Needs** — [`moving_objects.h`](moving_objects.h.md)
**Used by** — [`moving_objects_dynamic.cpp`](moving_objects_dynamic.cpp.md) · [`moving_objects_dynamic_collision.cpp`](moving_objects_dynamic_collision.cpp.md) · [`moving_objects_static.cpp`](moving_objects_static.cpp.md)
**Tier floor** — T3: constants and one distance test

## Purpose

A private tuning header shared by the three implementation files of the avoidance system,
so that all of them sample the future on the same grid. Its value to a rebuild is that
these six numbers, together, *are* the character of the system: change them and creatures
visibly bunch up, or dither, or walk through each other.

## State

```text
step_to_check       = 0.5  real   # metres between samples along a predicted path
time_to_check       = 1.0  real   # seconds of future the prediction covers
wait_radius_factor  = 2.0  real   # a waiting creature's footprint is doubled
                                  # in both horizontal axes, never in height
inertia_time_ms     = 500  int    # declared; see the note
additional_radius   = 2.0  real   # extra metres of search once a collision is known
max_linear_velocity = 10.0 real   # assumed worst-case speed of any other creature
```

**Invariants** — the prediction horizon and the sample step together fix the sample count
at *speed × horizon ÷ step*, so a creature walking at two metres a second is tested at
four points along its next second. One second is roughly the time it takes a walking
creature to stop and start again, which is why deciding any later would not help and
deciding any earlier would make creatures yield to collisions that never happen.

A waiting creature's footprint is enlarged **horizontally only**, and only while it waits.
That makes a stopped creature a bigger obstacle than a moving one, which is what stops two
creatures from both stopping nose to nose and then both deciding to move again.

The worst-case speed is used to size the proximity query: a creature must find everyone who
could possibly reach it within the horizon, and since the index stores positions rather
than velocities, the only safe bound is "my own speed plus the fastest anything moves".
Ten metres per second is above every walking or running creature in the shipped data; a
rebuild adding a faster creature must raise it or that creature will be missed.

The inertia interval is defined here and read nowhere. The avoidance record does keep the
wall-clock time at which its decision last changed — which is exactly the value this
interval would be compared against — so the intent is legible and the mechanism was never
finished. A rebuild that wants stable decisions should implement it: refuse to change a
creature's action within this many milliseconds of its last change.

## `collided`

**Contract** — the one shared predicate: two circles overlap when the distance between their
centres is at most the sum of their radii. Given an object and a (position, radius) pair, it
answers whether the object's own footprint reaches that circle.

**Notes** — a circle test, not a box test, and inclusive at the boundary. The dynamic side
uses oriented boxes instead (see
[`moving_objects_dynamic_collision.cpp`](moving_objects_dynamic_collision.cpp.md)); the
static side uses this, because level furniture has a meaningful radius and no meaningful
heading.
