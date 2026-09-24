# src/xrGame/interactive_motion.h

> Declares the base for a death animation that is allowed to collide with the world and bail out into ragdoll when it cannot continue.

**Needs** — [`interactive_motion.cpp`](interactive_motion.cpp.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`imotion_position.cpp`](imotion_position.cpp.md) · [`imotion_position.h`](imotion_position.h.md) · [`imotion_velocity.cpp`](imotion_velocity.cpp.md) · [`imotion_velocity.h`](imotion_velocity.h.md) · [`interactive_motion.cpp`](interactive_motion.cpp.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in [`interactive_motion.cpp`](interactive_motion.cpp.md) and
specialized by [`imotion_position.h`](imotion_position.h.md) and
[`imotion_velocity.h`](imotion_velocity.h.md).

An authored death animation looks far better than a ragdoll — but only if the body has room
to play it. This is the base of the mechanism that plays one anyway and **abandons it mid-way
into a ragdoll** the moment the body hits something the animation did not anticipate.

## State

```text
RECORD InteractiveMotion
  motion : MotionID          # the death animation to play
  shell  : PhysicsShell      # the body it drives
  angle  : real              # an authored heading offset applied to the motion
  flags  : bit set:
    use_death_motion   # this mechanism is active at all
    switch_to_ragdoll  # the abort signal — set by collision, by the move step failing,
                       # or by the animation reaching its end
    started            # state_start has run and state_end has not
    not_played         # declared and unused
```

**Invariants** — the abort flag is set from three unrelated places (the animation's end
callback, the collision test, the movement step) and is consumed in one. That fan-in is the
design: any of them can decide the animation cannot continue, and none needs to know about
the others.

The started flag exists so that destruction can run the teardown exactly once even when the
motion is abandoned early. The destructor asserts every flag is clear, which is how "you must
call destroy" is stated.

## Exported units

- `setup`, by motion name or by motion identifier — binds a motion, a body and an angle, and
  arms the mechanism. The named form resolves through the skeleton.
- `play` — starts the animation, with the abort callback installed at its end, then runs the
  subclass's start.
- `update` — one frame. Collide, and if not aborted, move; if the move aborted, release.
- `is_enabled` — whether the mechanism is armed.
- `destroy` — teardown, safe whether or not the motion started.
- `switch_to_free` — release the body into ordinary ragdoll simulation.
- `move_update` and `collide` — demanded of the subclass; the two halves that differ between
  the position-driven and velocity-driven variants.
- `state_start` and `state_end` — overridable, with a base implementation.

## `interactive_motion_diagnostic`

**Contract** — a debug-only trace naming the motion, the object and its model. Compiled out of
shipped builds.

**Notes** — a free `destroy` helper exists that tears down and releases a motion in one call,
tolerating an absent one. It exists because the teardown and the release must always happen
together and in that order.
