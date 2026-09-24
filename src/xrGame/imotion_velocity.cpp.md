# src/xrGame/imotion_velocity.cpp

> Plays a death animation by asking the physics solver to reach each animated pose through velocity, so the body can be stopped by the world instead of passing through it.

**Needs** — [`imotion_velocity.h`](imotion_velocity.h.md) · [`interactive_motion.h`](interactive_motion.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`imotion_velocity.h`](imotion_velocity.h.md)
**Tier floor** — T2: drives a rigid-body solver through velocity targets

## Purpose

There are two ways to make a physics body follow an animation. Setting its **position** every
step reproduces the animation exactly and lets the body pass through walls. Setting its
**velocity** so that the solver *would* reach the animated pose leaves the solver free to
refuse — to be stopped by a wall, to slide along a slope — and so produces a death animation
that interacts with the world. This file is the second way; its sibling
[`imotion_position.cpp`](imotion_position.cpp.md) is the first.

The cost of the velocity approach is that the body can fall arbitrarily far behind the
animation, and the mechanism must notice when it has and give up.

## State

`Stateless` — it operates the base's record.

## `state_start`

**Contract** — runs the base start, and if the mechanism is armed, **switches gravity off**
on the body.

**Invariants** — gravity must be off for the whole of the animation. The velocities computed
to reach each animated pose already account for the vertical motion the animator authored;
adding gravity on top makes the body sink faster than the animation and immediately fall
behind. The switch is conditional on the mechanism actually being armed, so a start on a
disarmed motion does not leave gravity off on a body nobody will restore it for.

**Notes** — three further configuration steps — raising the body's dynamic limits, resetting
its scales, and removing air resistance — are present in the source and disabled. They belong
to a tuning pass that was abandoned; a rebuild should not reinstate them without measuring.

## `move_update`

**Contract** — one frame. Asks the solver for velocities that would reach the current animated
pose within one frame; if it cannot, raises the abort flag. Then performs a transform dance
whose only purpose is a side effect.

```text
FUNCTION move_update()
  reached = body.anim_to_velocity_state(frame_time,
                                        linear_limit  = 2  * default_linear_limit,
                                        angular_limit = 10 * default_angular_limit)
  IF NOT reached THEN abort            # the body cannot keep up: let go into ragdoll

  saved = body.transform
  body.interpolate_global_transform(into body.transform)   # for its SIDE EFFECT
  body.transform = saved                                   # then put it back
```

**Invariants**:

- **The angular limit is five times more generous than the linear one** (ten versus two times
  the defaults). A death animation's rotations are fast and large — a body spinning as it
  falls — while its translations are modest, and a body that cannot rotate fast enough to
  follow aborts immediately. The asymmetry is what makes the mechanism usable at all.
- **The transform is saved, overwritten and restored.** The overwrite is not wanted; the
  interpolation call's internal bookkeeping is. It refreshes the body's interpolation state
  so that a later read — in particular the release into ragdoll — gets a current interpolated
  transform rather than a stale one. A rebuild whose physics interface exposes that
  bookkeeping separately should call it directly and delete the dance.

## `state_end`

**Contract** — runs the base end, then makes a **final** velocity-matching call with limits ten
times the defaults on both axes, then switches gravity back on.

**Invariants** — the order matters in both directions. The final velocity call happens *before*
gravity is restored, so the body enters ragdoll carrying the animation's last velocity rather
than one already perturbed by a frame of gravity — that is what makes the hand-off look
continuous. And the limits are raised for this one call because the point is no longer to
follow the animation faithfully but to leave the body moving in roughly the right direction;
refusing the last velocity because it exceeds a limit would drop the body dead-still.

Gravity must be restored unconditionally, including on an aborted motion, or the ragdoll
floats.

## `collide`

**Contract** — deliberately empty. The velocity-driven variant needs no separate collision
test: the solver's own refusal to reach the target pose *is* the collision signal, and it is
already reported through the move step.

**Invariants** — a rebuild must not "fill in" a collision test here. Doing so would abort on
resting contact, which the velocity path handles correctly by simply not reaching the pose.
