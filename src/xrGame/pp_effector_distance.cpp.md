# src/xrGame/pp_effector_distance.cpp

> A screen-effect controller that ramps its post-process effect up as the viewer approaches a source and turns it off at the outer edge.

**Needs** — [`pp_effector_distance.h`](pp_effector_distance.h.md) · [`pp_effector_custom.h`](pp_effector_custom.h.md)
**Used by** — reached through its declarations in [`pp_effector_distance.h`](pp_effector_distance.h.md); callers name that, not this file.
**Tier floor** — T3: two configuration reads and one linear interpolation

## Purpose

Some effects on the player's view — the visual signature of a nearby anomaly or artefact —
should be strongest at the source and vanish at a stated distance. This file supplies the
one rule that converts *how far away am I* into *how strong is the effect*, and hands it to
the generic controlled-effector machinery in
[`pp_effector_custom.h`](pp_effector_custom.h.md), which owns the start/stop lifecycle and
the actual per-frame blending of post-process parameters.

The split is worth keeping: the generic side knows nothing about distance, and every other
ramp rule (time-based, health-based) can be written the same way.

## State

```text
RECORD DistanceEffectorController        # extends the generic controlled-effector state
  radius_min_fraction : real   # fraction of radius at which the effect reaches full strength
  radius_max_fraction : real   # fraction of radius at which the effect is gone
  radius              : real   # the source's influence radius, pushed in by the owner
  distance            : real   # current viewer-to-source distance, pushed in by the owner

  # invariant: radius_min_fraction <= radius_max_fraction, checked at load.
  #            Both are fractions of `radius`, not absolute distances, so one
  #            configuration section describes anomalies of every size.
```

`radius` and `distance` are *pushed* by whoever owns the source each frame; the controller
never queries the world itself. That keeps it usable by anything that can measure a
distance.

## `load`

**Contract** — reads `radius_min` and `radius_max` (both fractions in `[0,1]`) from the
named configuration section, after the generic controller has read the post-process
parameters it blends toward. Fails loudly if the two are out of order, since an inverted
band would make the ramp divide by a negative number and invert the effect.

## `check_start_conditions` · `check_completion`

**Contract** — the effect starts when the viewer is inside the outer edge
(`distance < radius * radius_max_fraction`) and completes when it is outside it. The two
tests are exact complements on the same threshold, so there is no hysteresis: an object
sitting exactly at the boundary will start and stop the effect on alternate frames.

## `update_factor`

**Contract** — recomputes the effect's strength each frame while active and writes it to
the effector.

```text
FUNCTION update_factor()
  outer = radius * radius_max_fraction
  inner = radius * radius_min_fraction
  factor = (outer - distance) / (outer - inner)   # 0 at the outer edge, 1 at the inner
  factor = clamp(factor, 0.01, 1.0)
  effector.set_factor(factor)
```

**Notes** — the low clamp is `0.01`, not zero. A factor of exactly zero would be
indistinguishable from "not running", and the generic effector uses the factor as a blend
weight; keeping it barely positive means the effect fades to invisible while remaining
installed, so that the completion test — not a degenerate weight — is what removes it.

## `create_effector`

**Contract** — the factory hook: produces a controlled effector bound to this controller
and to the post-process parameter set read at load. Exists so that the generic controller
can build its effector without knowing the concrete subclass.

**Notes** — nothing in the shipped codebase constructs this class. It is a complete,
working ramp rule with no caller, which most likely means an effect that was cut; a rebuild
may drop it, but the pattern it demonstrates (push distance in, get a blend factor out) is
the one every other view effect follows.
