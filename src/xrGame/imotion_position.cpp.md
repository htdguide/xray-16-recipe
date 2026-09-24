# src/xrGame/imotion_position.cpp

> Plays a death animation by teleporting the body to each animated pose, but only after speculatively advancing the animation and checking that the poses ahead do not drive it into the world.

**Needs** — [`imotion_position.h`](imotion_position.h.md) · [`interactive_motion.h`](interactive_motion.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`animation_utils.h`](animation_utils.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/ExtendedGeom.h`](../xrPhysics/ExtendedGeom.h.md) · [`xrPhysics/MathUtils.h`](../xrPhysics/MathUtils.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/buffer_vector.h`](../xrCore/buffer_vector.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`imotion_position.h`](imotion_position.h.md)
**Tier floor** — T1: takes over the skeleton's evaluation pipeline, speculatively advances animation state and rolls it back, and allocates the rollback buffer on the stack inside a per-frame loop

## Purpose

The sibling variant, [`imotion_velocity.cpp`](imotion_velocity.cpp.md), lets the physics
solver decide whether the body can reach each animated pose. That is robust but imprecise —
the body lags the animation and the death reads as mushy. This variant does the opposite: it
**places the body exactly on the animated pose**, which looks exactly like the authored
animation, and pays for it by having to detect interpenetration itself.

Detection alone is not enough, because by the time a pose interpenetrates it is already too
late — the body is inside a wall. So the mechanism **speculates**: it advances the animation
a short distance into the future, tests those poses, and then *rewinds the animation state* to
where it really was. If the future looks clear the animation continues; if the penetration is
deepening, or is still deep after the look-ahead, the animation is abandoned and the body
becomes a ragdoll.

That speculation is the whole file. Everything else — the pipeline takeover, the bone
suppression, the blend save and restore, the stack-allocated rollback buffer — exists to make
it possible to run the animation forward and put it back.

## State

```text
RECORD PositionMotion
  time_to_end           : real
  blend                 : optional<Blend>
  saved_visual_callback : optional<Callback>
  shell_has_history     : bool
  update_intercepted    : bool

# module-scope: written by the contact filter, read by every collision test
deepest_penetration : real
```

## Constants

```text
max_collide_timedelta = 0.02     # seconds: the longest animation step that may be taken
                                 # before re-testing for collision
end_delta             = 0.01     # = half of the above; the final advance at hand-off
collide_advance_delta = 0.04     # = twice the above; how far the look-ahead reaches
depth_resolve         = 0.01     # metres: penetration deeper than this is a collision
```

**Invariants** — the three time constants are derived from one another, and a rebuild should
keep them so. The step bound is chosen small enough that a body moving at animation speed
cannot cross a thin wall within one step; the look-ahead is two steps because one step is not
enough to tell a deepening intersection from a grazing one; the final advance is half a step
so that the hand-off pose is unambiguously past the last tested one.

The penetration threshold of a centimetre, against the sibling mechanism's five, is tight
because here the pose is placed exactly — there is no solver slop to tolerate.

## `state_start` — taking over the pipeline

**Contract** — seizes control of the skeleton, the body and the animation system, in an order
where every step depends on the previous one.

```text
FUNCTION state_start()
  base.state_start()

  saved_visual_callback = skeleton.update_callback
  skeleton.update_callback = none          # the owner's post-pose hook must not run while
                                           # we are speculatively advancing the pose
  animation.tracks_update_hook = this      # INTERCEPT: the animation system no longer
                                           # advances its own tracks

  blend = the playing full-body blend whose end callback is ours
  REQUIRE blend is non-looping
  time_to_end = (blend.total - one_sample - epsilon - blend.current) / blend.speed

  body.add_contact_filter(record_deepest_penetration)

  IF the mechanism is not armed THEN RETURN   # the filter and interception stay: see below

  owner.processing_activate()              # the owner must be updated every frame now
  body.disable()                           # the SOLVER is off; we place the body ourselves
  body.enabled_callbacks(false)
  install_root_bone_callback()             # applies the authored heading offset
  body.transform = owner.transform
  intercept_updates(true)
  suppress_bone_calculation(true)

  collide_not_move()                       # is the body ALREADY intersecting?
  IF aborted THEN
    switch_to_free() ; mark not-played ; RETURN

  move(one frame)                          # the first frame runs here, not next frame
  IF aborted THEN switch_to_free()
```

**Invariants**:

- **The body's solver is disabled and its transform is set from the owner**, not the other way
  round. For the duration of this animation the body is not simulated at all; it is placed.
- **The owner is put into per-frame processing.** A dead creature would otherwise be updated
  at the scheduler's degraded rate, and the sub-stepping needs a real frame time.
- **The immediate-collision test runs before any advance.** A creature that died already
  wedged into geometry is released to ragdoll without ever playing a frame of the animation,
  and is marked as not-played so the caller knows the authored death did not happen.
- **The first frame is moved inside the start**, so a death animation that is immediately
  impossible fails within the same frame the creature died rather than a frame later.
- The time remaining subtracts **one animation sample plus an epsilon** from the total. The
  last sample of a non-looping animation is the pose the blend holds at; advancing into it
  would leave the mechanism with no pose left to advance to while the abort has not fired.
- The time remaining is divided by the blend's **speed**, converting animation time into real
  time. Everything downstream is in real seconds.

**Notes** — the search for the blend is by matching the end callback and requiring the blend
to cover the whole body rather than a part. There is no handle to the blend at play time
because the animation system allocates it internally. A rebuild whose play call returns a
handle should use it.

## `state_end` — handing back

**Contract** — reverses everything, in an order where several steps are load-bearing.

```text
FUNCTION state_end()
  base.state_end()
  owner.processing_deactivate()
  body.enable()                                # the solver takes over again
  body.force = 0 ; body.torque = 0             # discard anything accumulated while placed

  body.anim_to_velocity_state(end_delta, limits x10)   # seed the ragdoll's velocities from
                                                       # the last animation step

  body.remove_contact_filter()
  intercept_updates(false)
  suppress_bone_calculation(false)
  skeleton.update_callback = saved_visual_callback
  remove_root_bone_callback()

  saved = the set of bones holding an authored fixed transform
  body.enabled_callbacks(true)                 # this REPLACES those bones' callbacks
  restore(saved)                               # so they must be reinstated afterwards

  IF the skeleton's root bone is not bone zero THEN
    hide bone zero, and hide the named biped root if it is a different bone
  skeleton.invalidate_bone_cache()
  skeleton.calculate_bones(forced)
```

**Invariants**:

- **Forces and torques are zeroed before the velocity seeding.** While the body was placed
  rather than simulated, the solver may still have accumulated contact forces; releasing with
  those applied throws the corpse.
- **The velocity seeding is over the `end_delta` interval**, not over a frame. It converts the
  last small animation advance into the velocity the ragdoll starts with, which is what makes
  the hand-off look like one continuous motion instead of a body stopping and then falling.
- **Re-enabling the body's callbacks destroys the authored bone fixes**, so they are collected
  beforehand and re-applied afterwards. A bone fix pins a bone to an authored transform (a
  weapon in a hand, a prosthetic); losing it leaves the bone flapping. This save/restore pair
  is a genuine ordering trap and the source has no comment on it.
- **The root-bone hiding** handles models whose skeleton root is not the first bone. Such a
  model has a redundant bone zero (and sometimes a redundant named biped root) that the
  animation drove and the ragdoll will not; left visible they render as a stretched shard of
  mesh. Both are collapsed to identity and hidden.
- The final invalidate-and-recalculate must be immediate, for the same reason as in
  [`interactive_motion.cpp`](interactive_motion.cpp.md): the ragdoll seeds itself from the
  bone positions.

## `move` — one frame, sub-stepped

**Contract** — advances the animation by one frame's worth of time, in steps no longer than
the collision bound, stopping at the first step that aborts.

```text
FUNCTION move(dt) -> real     # returns how much animation time was actually advanced
  steps = 1 ; step_dt = dt
  IF dt > max_collide_timedelta THEN
    steps   = ceil(dt / max_collide_timedelta)
    step_dt = dt / steps                     # EQUAL steps, not a remainder

  advanced = 0
  REPEAT steps TIMES
    IF not aborted THEN
      save every blend's state onto the stack
      advanced += motion_collide(step_dt)

    IF aborted THEN
      restore every blend's state          # undo the failed step's advance
      time_to_end -= advanced
      recompute the pose
      place the body on the animated bone positions, keeping its motion history
      shell_has_history = true
      advanced += advance_animation(end_delta)   # one last small advance into the pose
                                                 # the ragdoll will start from
      BREAK
  RETURN advanced
```

**Invariants**:

- **The frame is divided into equal steps**, not into whole steps plus a remainder. A short
  final step would test a shorter span than the others and could miss an intersection the
  bound was chosen to catch.
- **On abort the blends are rolled back and then the animation is advanced by `end_delta`
  anyway.** The body must not be left on the pose that intersected; it is left on a pose
  slightly past the last good one, which is where the velocity seeding in the end hook will
  measure from.
- **The body is placed with its motion history preserved** on the abort path and the history
  flag is then set. The history is what lets the physics layer compute a velocity from
  successive placements; discarding it would hand the ragdoll a zero velocity.
- The blend state is saved into a buffer allocated **on the stack**, sized from the live blend
  count, every sub-step. That is a per-frame, per-sub-step allocation in the hot path; it is
  on the stack precisely so it costs nothing. A rebuild should use a reusable buffer rather
  than reproduce the stack allocation.

## `motion_collide` — the speculation

**Contract** — one sub-step. Advance, test, and if the test fails, look ahead two further
steps before deciding. Always leaves the animation at the position the sub-step intended,
regardless of how far it speculated.

```text
FUNCTION motion_collide(dt) -> real
  advanced = collide_animation(dt)          # advance, pose, place, collide

  IF time_to_end < max_collide_timedelta + end_delta THEN
    abort ; RETURN advanced                 # the animation is about to run out anyway

  IF deepest_penetration <= depth_resolve THEN RETURN advanced    # clear

  # --- we are intersecting. Look ahead. -----------------------------------
  save every blend's state
  depth_before = deepest_penetration
  advanced += collide_animation(collide_advance_delta)

  IF deepest_penetration > depth_before THEN
    abort                                   # deepening: the animation is driving INTO it
  ELSE
    depth_before = deepest_penetration
    advanced += collide_animation(collide_advance_delta)
    IF deepest_penetration > depth_resolve THEN
      abort                                 # still stuck after two look-ahead steps

  restore every blend's state               # UNDO the look-ahead
  time_to_end += (dt - advanced)            # ... and the animation clock with it
  advanced = dt                             # the caller only ever advanced by dt
  recompute the pose
  place the body on the animated positions, CLEARING its motion history
  RETURN advanced
```

**Invariants**, and this is the core of the file:

- **A single deep contact is not a failure.** An animation legitimately brushes the ground, a
  wall, a corpse. The decision is made on the *trend*: if the next look-ahead step is deeper,
  the animation is pushing into the obstacle and cannot continue. If it is shallower, the
  animation is moving away and is allowed to carry on.
- **Two look-ahead steps, not one.** A pose can be momentarily deeper and then clear — a limb
  swinging past a corner. The second step is what distinguishes that from a genuine wedge.
  The second test is against the absolute threshold rather than the trend, because by then
  four hundredths of a second have passed and anything still intersecting is stuck.
- **The look-ahead is fully undone.** Blend states are restored, the animation clock is given
  back exactly the time the speculation consumed, and the reported advance is the sub-step's
  own `dt` and nothing more. The caller must be unable to tell that speculation happened.
- **The motion history is CLEARED on this path** and kept on the abort path in `move`. Here the
  body has been placed at poses that were then rolled back; a velocity computed across that
  sequence would be nonsense. There the placements are the real ones and the velocity is
  wanted.
- **Running out of time also aborts**, with the same threshold as one step plus the final
  advance. The animation ending and the animation failing are handled identically, which is
  the same unification the base class makes.

## `collide_animation` and `advance_animation`

**Contract** — `advance_animation` moves the animation clock forward, advances the tracks,
evaluates the whole skeleton, and re-disables the body's solver. `collide_animation` does that
and then places the body on the resulting bone positions, zeroes the recorded penetration, and
runs a full collision pass to fill it in.

**Invariants** — the penetration must be zeroed immediately before the pass and read
immediately after, because it is module-scope state shared by every instance. The collision
pass's *contacts* are discarded; only the depth is wanted.

The body's solver is re-disabled after every pose evaluation because evaluating the pose can
re-enable it.

## `collide_not_move`

**Contract** — tests whether the *current* pose intersects, without committing to any advance:
saves the blends, runs a half-step collision test, restores. Used once, at the start, to reject
a death that begins already wedged.

## `force_calculate_bones`

**Contract** — evaluates the whole skeleton with the suppression lifted, then re-suppresses.
If the owner had a post-pose hook, it is invoked — with the skeleton's bone root temporarily
set to bone zero and restored afterwards.

**Invariants** — the temporary root change exists because the owner's hook expects to walk the
whole skeleton from its true root, and the death animation may have left the bone root
pointing elsewhere. Restoring it afterwards is not optional; the pose evaluation depends on it.

## Bone-calculation suppression

**Contract** — sets an overwrite flag on every bone **except the real root**, which tells the
skeleton's evaluation not to recompute that bone from the animation. Bones already carrying a
callback payload are skipped.

**Invariants** — the root is excluded because the root *is* driven here, by the root callback
below. Bones with a payload are skipped because they belong to someone else — a bone fix, an
aiming turret — and stealing their flag would break them and leave them broken, since the
suppression is cleared en masse at the end.

**Notes** — the code warns in debug builds when a bone's flag is already in the state being
set, which is the symptom of two mechanisms suppressing the same skeleton. That warning is the
only documentation that the suppression is not re-entrant.

## The root-bone callback

**Contract** — invoked while the skeleton's root bone is being built. Rebuilds the root's
transform from the animation's key channels, **replacing the death animation's own rotation
with a rotation about the vertical by the authored angle**.

```text
FUNCTION on_root_bone(bone)
  IF updates are not intercepted THEN RETURN
  keys = the per-blend rotation and position keys for this bone
  FOR EACH key channel
    IF the key's blend is OUR death animation THEN
      key.rotation = rotation about the vertical axis by `angle`
  bone.transform = build from keys
  REQUIRE the result is a valid transform
```

**Invariants** — this is how a death animation is aimed. The animator authored the death
facing one direction; the game needs the corpse to fall in the direction the creature was
facing, or away from the shot. Rather than re-author every death for every heading, the root's
rotation is **overwritten** — not composed with — by a pure heading rotation, and the rest of
the animation plays relative to it.

Only the death animation's own key is replaced. Other blends contributing to the root keep
their rotations, so a death blending out of a walk still inherits the walk's root motion.

The result is validated rather than trusted, because a malformed key set produces a
non-finite transform that would propagate silently through the whole skeleton and into the
physics body.

## `move_update` and the tracks hook

**Contract** — the per-frame entry the base calls: lift the interception, invalidate and
recompute the pose, re-arm the interception.

The interception hook itself answers "I handled it" and routes the animation system's advance
into `move` — but only while armed. When disarmed it answers false and the animation system
advances normally.

**Invariants** — the momentary lift in `move_update` is what actually produces the visible
pose: with the interception armed, every pose evaluation goes through the sub-stepped,
speculative path; with it lifted for exactly one evaluation, the skeleton is computed normally
from the animation state the sub-stepping left it in. A rebuild must keep the two paths
distinct — the speculative advance and the presented pose are not the same computation.

## Debug instrumentation

**Notes** — four debug switches draw the skeleton, the velocities, the collision geometry and a
per-step trace of penetration depths, and a diagnostic prints the motion name, the blend times
and the colliding object at every decision point. They are compiled out of shipped builds. The
density of the instrumentation is itself informative: the speculation's decisions are not
inspectable any other way.
