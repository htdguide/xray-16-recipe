# src/xrGame/MercuryBall.cpp

> An artefact that will not sit still: every so often it gives itself a random horizontal shove and rolls somewhere else.

**Needs** — [`MercuryBall.h`](MercuryBall.h.md) · [`Artefact.h`](Artefact.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`MercuryBall.h`](MercuryBall.h.md)
**Tier floor** — T3: a timer and an impulse

## Purpose

Most artefacts lie where they fell. This one rolls, which is its whole identity as a game
object: it is harder to pick up and it wanders away from where the player last saw it. The
behaviour is one rule applied on a timer.

## State

```text
RECORD MercuryBall
  time_last_update : int       # when the last roll decision was made
  time_to_update   : int       # ms between decisions;   default 1000
  impulse_min      : real      #                         default 45
  impulse_max      : real      #                         default 90
```

All three tuned values are read from the artefact's configuration section and the
constructor's values are only a fallback for a section that omits them.

## `Load`

**Contract** — reads the decision interval and the impulse range from the section, on top of
the inherited artefact load. All three keys are required; a section missing one fails.

## `UpdateCLChild`

**Contract** — the per-frame behaviour hook. When the artefact is a free object in the world it
periodically shoves itself; when it is carried it simply follows its owner's transform.

```text
FUNCTION UpdateCLChild()
  IF visible AND a physics body exists THEN
    IF now - time_last_update > time_to_update THEN
      time_last_update = now
      # Only about two rolls in five actually happen, so the motion is irregular rather
      # than metronomic. A ball that shoved itself on every tick would read as driven.
      IF random_unit() > 0.6 THEN
        dir = (random in [-0.5, 0.5], 0, random in [-0.5, 0.5])   # horizontal only
        impulse = random in [impulse_min, impulse_max]
        apply impulse (dir, impulse * frame_time * body mass)
      END IF
    END IF
  ELSE IF it has a parent THEN
    transform = parent's transform      # carried: no physics, just follow
  END IF
```

**Invariants** — the impulse is scaled by the **frame time** and by the body's **mass**.
Scaling by mass makes the tuned numbers an acceleration rather than a force, so the same
configuration produces the same motion whatever the artefact's mass. Scaling by frame time is
questionable here — the impulse is applied once per decision interval, not once per frame, so
frame rate leaks into how far the ball rolls. A rebuild should scale by the decision interval
instead; the recipe cannot establish whether the original intended that.

The direction is deliberately horizontal: the vertical component is zero, so the ball rolls
rather than hops.

**Notes** — the carried branch setting the transform directly from the parent is the standard
shape for an item with a physics body that is currently owned: the body is not stepped, and
the visual is slaved to the owner. The condition that selects between the two branches is
visibility plus the existence of a body, not ownership, which means an invisible free ball
also takes the parent branch and does nothing — correct, if indirect.
