# src/xrGame/ik_anim_state.cpp

> Reads the playing animation's footfall markers to decide, for one limb, whether the foot is planted, whether it may be pinned to the ground, and whether two animations are blending across that decision.

**Needs** — [`ik_anim_state.h`](ik_anim_state.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`xrCore/Animation/Motion.hpp`](../xrCore/Animation/Motion.hpp.md)
**Used by** — [`ik_anim_state.h`](ik_anim_state.h.md)
**Tier floor** — T2: interval arithmetic over animation time

## Purpose

Inverse kinematics can only place a foot on the ground while the animation says that foot is
*supposed* to be on the ground. An animator marks those spans: each motion carries, per limb,
a set of time intervals during which that limb is planted. This file turns those marks, plus
the state of the animation blend, into four booleans the solver acts on.

The hard part is not reading the marks — it is deciding what to do while **two** animations
are blending, because the outgoing one may say planted and the incoming one may not. Getting
that wrong produces the classic artifact of a foot that jumps as a walk transitions to an
idle. Most of this file is the truth table for that case.

## State

```text
RECORD AnimState
  is_step       : bool    # the animation says this limb is planted now
  do_glue       : bool    # the foot may be pinned to where it was, not just placed
  is_idle       : bool    # the governing animation is an idle
  is_blending   : bool    # two animations are crossfading
  current_blend : optional<Blend>   # the blend this state was last computed against
```

**Invariants** — `glue` implies the foot's *world position* is held fixed across frames;
`step` only means the foot should be placed on the ground this frame. The two are different
and the blending case can produce step without glue.

## Animation time

Three small pieces of arithmetic the rest depends on:

```text
FUNCTION blend_time(blend) -> real
  # where inside the motion the blend currently is, wrapped into one cycle
  t = blend.current_time / blend.total_time
  RETURN (t - floor(t)) * blend.total_time
```

**Invariants** — the wrap is what makes looping animations work: a blend's current time runs
past the motion's length, and the marks are authored within one cycle. The subtraction of the
floor rather than a modulus keeps the result exact at the cycle boundary.

```text
FUNCTION is_inside(interval, value) -> bool
  IF interval.start < interval.end THEN RETURN start < value < end
  ELSE RETURN value > start OR value < end      # the interval WRAPS the cycle end
```

**Invariants** — an interval whose end precedes its start wraps around the loop point. That
is how an animator marks a footfall that straddles the seam of a looping cycle, and it is a
real case in the shipped data.

```text
FUNCTION time_to_next_mark(blend, marks) -> real
  now = blend_time(blend)
  t = marks.time_to_next_mark(now)
  IF t is finite THEN RETURN t                        # a mark later in this cycle
  t = marks.time_to_next_mark(just_after_zero)
  IF t is finite THEN RETURN t + blend.total_time - now   # wrap to the first mark of the
                                                          # next cycle
  RETURN blend.total_time - now                       # no marks at all: the cycle's end
```

**Invariants** — the second probe starts at a small epsilon rather than at zero, because a
mark exactly at time zero must be found and a query *at* a mark's time does not count as
"next".

## `update`

**Contract** — recomputes the four flags for one limb from the currently playing blend.
Everything starts false; an absent blend clears the remembered blend and returns with all
four false.

```text
FUNCTION update(skeleton, blend, limb_index)
  is_step = is_idle = do_glue = is_blending = false
  IF no blend THEN current_blend = none ; RETURN

  new_motion = skeleton.motion_definition(blend.motion)
  IF new_motion has no marks for this limb THEN RETURN     # the limb is unmarked: the
                                                           # solver leaves it alone

  IF we are crossfading (see below) THEN
    is_blending = true
    old_motion  = skeleton.motion_definition(current_blend.motion)
    old_planted = old_motion has marks for this limb AND current_blend is inside one
    new_planted = blend is inside one of new_motion's marks
    is_idle     = BOTH motions are idles
    any_idle    = EITHER motion is an idle

    do_glue = (old_planted AND new_planted) OR (any_idle AND new_planted)
    is_step = (NOT any_idle AND (old_planted OR new_planted)) OR (any_idle AND do_glue)
  ELSE
    is_step       = blend is inside one of new_motion's marks
    current_blend = blend            # only the non-blending path adopts the new blend
    is_idle       = new_motion is an idle
    do_glue       = true
```

**Invariants** — the truth table is the substance:

- **Both planted** → glue. The foot is planted in both animations, so it is genuinely
  planted and may be pinned.
- **Neither is an idle, one planted** → step but no glue. Placing the foot on the ground is
  right, pinning it is not, because the other animation is going to move it.
- **One is an idle and the new one says planted** → glue. Transitions into and out of an idle
  are the ones that visibly slide, and idles are stationary, so pinning is safe and is what
  removes the artifact.
- **One is an idle and only the *old* one says planted** → neither step nor glue. The limb is
  being released; holding it would fight the incoming animation.

Note the asymmetry: with an idle involved, the decision keys off the **new** animation only.
That directionality is deliberate and is why walk-to-idle and idle-to-walk look different.

The remembered blend is updated **only on the non-blending path**. While a crossfade is in
progress the state keeps comparing against the animation that was playing before it started,
which is what makes the crossfade a two-sided comparison at all. A rebuild that updates the
remembered blend every frame collapses the blending case to the simple one and loses every
transition fix above.

### Deciding that a crossfade is in progress

```text
FUNCTION crossfading(current, new) -> bool
  RETURN current exists
     AND current is not a freed slot
     AND current is not the same blend as new
     AND new.blend_amount < new.blend_power - epsilon
```

**Invariants** — the last condition is what ends a crossfade: once the incoming blend has
reached full weight there is nothing to reconcile and the simple path takes over, adopting it
as the new remembered blend. The freed-slot test guards against a remembered blend whose
storage has been recycled for a different animation.

## `time_step_begin`

**Contract** — how long until this limb's next footfall begins, for a given blend. Answers
false, with a zero time, when the question is meaningless: the limb has no marks in this
motion, the motion is an idle, or the mark set is empty.

**Invariants** — idles are excluded explicitly even when they carry marks. An idle has no
footfalls to anticipate, and returning its cycle length would make the caller schedule work
that never becomes relevant.

**Notes** — the answer is used to decide *when* to next reconsider a limb rather than what to
do with it, so an overestimate costs smoothness and an underestimate costs time. The fallback
when a motion has no marks — the time remaining in the cycle — is therefore the conservative
choice rather than an arbitrary one.

## `auto_unstuck`

**Contract** — always true. The conditions under which it would have been false (only while
idle or blending) are preserved in the source but disabled: a foot that has been pinned into
geometry is always released, in every state.
