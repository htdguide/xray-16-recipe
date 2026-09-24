# src/xrGame/ai/monsters/control_path_builder_base_inline.h

> The one-line setters of the path-builder base, and the one place the chapter's default path-following parameters are written down.

**Needs** — [`control_path_builder_base.h`](control_path_builder_base.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: field assignments and one set of defaults

## Purpose

Trivial setters, separated from the declaration only because C++ wanted them defined after
the class. The file would vanish in a rebuild but for one function.

## State

Stateless.

## `set_cover_params`, `set_use_covers`, `set_rebuild_time`, `set_distance_to_end`

**Contract** — field writes. The cover parameters are the minimum and maximum distance from
the target a cover point may be, the angular deviation allowed from the direction to it,
and the radius within which to search. The rebuild time is the minimum gap between replans
of a path that is still valid; the distance-to-end is how close along the path counts as
arrived.

## `set_generic_parameters`

**Contract** — the default path-following behaviour, applied by any state that does not
tune its own. Every number here is a constant in this file, not authored data, which makes
this the tuning that a rebuild must copy rather than read.

```text
FUNCTION set_generic_parameters()
  rebuild_time         <- 5000 ms   # a valid path is not replanned more often than this
  distance_to_path_end <- 3.0       # world units along the path that count as arrived
  use_covers           <- true
  cover_params(min 5.0, max 30.0, deviation 1.0 rad, radius 30.0)
```

**Notes** — five seconds between replans is what makes creatures commit to a route rather
than twitching as their enemy moves; it is also why a creature chasing a fast enemy visibly
cuts corners. Three units of arrival tolerance is roughly a body length. The cover band of
five to thirty units means the default cover search will never choose a point right on top
of the target nor one across the level. None of the five numbers has a recorded
derivation.
