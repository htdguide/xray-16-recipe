# src/xrGame/IKLimbsController.cpp

> Foot placement for one character: it hooks into the pose evaluation, corrects each leg onto the ground it is standing on, and lifts or drops the whole body so that every planted foot can reach.

**Needs** — [`IKLimbsController.h`](IKLimbsController.h.md) · [`ik/IKLimb.h`](ik/IKLimb.h.md) · [`ik_object_shift.h`](ik_object_shift.h.md) · [`pose_extrapolation.h`](pose_extrapolation.h.md) · [`ik_anim_state.h`](ik_anim_state.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`CharacterPhysicsSupport.h`](CharacterPhysicsSupport.h.md) · [`GameObject.h`](GameObject.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Motion.hpp`](../xrCore/Animation/Motion.hpp.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-bone transform arithmetic on the pose-evaluation path

## Purpose

An animation was authored on flat ground. The world is not flat. This controller closes
that gap for one character, and the shape of its solution is the thing to carry into a
rebuild:

**It runs as a callback on the skeleton's pose evaluation, not on the frame.** The
renderer evaluates the pose when it needs it; the correction has to happen after the
animation has produced a pose and before that pose is used, and the only hook that
satisfies both is the skeleton's own "pose is ready" callback. A rebuild that corrects on
the frame tick will correct last frame's pose.

**It corrects in two stages that must not fight.** Each foot is corrected onto its own
ground; but a foot on a step above the other cannot be reached without also moving the
*body*. So the controller first decides a single vertical shift for the whole character,
applies it to every bone, and only then solves each leg to its goal. Getting that order
wrong produces a character whose legs stretch instead of whose hips drop.

**It anticipates.** The body shift is not driven only by where the feet are now: a foot
that is *about to* plant contributes its predicted landing height, with the time until it
lands as the duration to move over. That is why a character walking down a slope leans
into the descent instead of snapping at each footfall.

## State

```text
RECORD LimbsController
  legs_blend   : optional<Blend>   # the animation currently driving the legs
  object       : GameObject
  limbs        : list<Limb>        # at most four; two by default
  object_shift : ShiftAnimator     # the character's vertical offset, with a target and a duration
  pose_history : Extrapolator      # recent transforms, so a future pose can be predicted
```

Invariants:

- The number of limbs comes from the model's own embedded configuration and defaults to
  two. Four is the hard maximum, and the per-frame working array is sized to it.
- The legs blend is dropped the moment the animation system frees its slot; every entry
  point re-checks it, because a blend pointer outliving its blend is the classic way this
  system crashes.
- Ground collision is attempted for a limb **only when the driving animation carries
  footstep marks**. An animation with no marks has no notion of when a foot is planted,
  so its feet are corrected without collision.

## `Create`

**Contract** — bind to a character, read the limb count from the model's embedded
configuration, build that many limbs, register the pose callback, and seed the pose
history from the current transform.

**Invariants** — the callback is registered and then, if other callbacks were already
registered, **swapped to the front**. Foot placement must run before anything else that
reads or modifies the pose, because everything else assumes the pose is final. This
one-line reordering is easy to miss and its absence produces subtle, intermittent
misplacement.

## `IKVisualCallback`

**Contract** — the static hook the skeleton calls when a pose has been evaluated. Recovers
the character from the skeleton's callback parameter, finds its controller, and runs the
correction.

## `Calculate` — the correction pass

**Contract** — the whole per-pose correction. Gathers each limb's calculation state,
suppresses the root bone's own callback for the duration, shifts the body, sets each
limb's goal, solves each limb's bones, re-evaluates the body shift for the next pose, and
restores the root callback.

**Invariants** — the ordering is the content of this function and every step depends on
its predecessor:

1. each limb's state is computed first, because the body shift is a function of all of
   them;
2. the **root bone's callback is disabled** across the whole pass. The root callback is
   what a character's own procedural motion (a physics-driven torso, a scripted lean)
   hangs off, and letting it run while the body is being shifted would apply the shift
   twice;
3. the body is shifted — every bone's translation moved vertically — *before* the goals
   are set, so each goal is expressed against the already-shifted skeleton;
4. goals are set for all limbs before any is solved, because a solve writes bone
   transforms that a later goal would otherwise read;
5. the shift is recomputed at the end with the solved poses, which is what feeds the
   next pose's starting point.

```text
FUNCTION calculate()
  drop the legs blend if the animation system has freed it
  FOR EACH limb: build its calculation state from the object transform and apply it
  save and disable the root bone's callback
  shift_object()                        # move every bone vertically by the current shift
  FOR EACH limb: set its goal
  FOR EACH limb: solve its bone chain onto that goal
  object_shift(cd)                      # re-decide the shift from the solved state
  restore the root bone's callback
```

## `ShiftObject`

**Contract** — apply the current vertical shift to the whole skeleton: add it to every
bone's world translation, then run each bone's own callback and add it again to the
render transform.

**Invariants** — the shift is applied **twice, to two different transform sets**: the
animation transforms and the render transforms, with the per-bone callbacks run between
them. That is because a bone callback (a turret's aim, a head look) writes the render
transform from the animation transform, and both must carry the shift.

**Notes** — this walks every bone of the skeleton twice per pose. For a character with a
hundred bones that is two hundred vector additions per pose evaluation, on every visible
character. The commented-out alternative — shifting only the root and re-deriving the
hierarchy — would be cheaper, and a rebuild should consider it; the reason it is not done
is that the callbacks have already run by then.

## `ObjectShift` — choosing between prediction and the static answer

**Contract** — decide how the body should move. If **not every** foot is currently
planted and a prediction is available, use the prediction; otherwise use the static
answer. Freezes the shift animation entirely while the game is paused.

**Invariants** — prediction is only consulted when at least one foot is in the air,
because a foot in the air is the only one whose landing can be anticipated. With every
foot planted there is nothing to predict and the static answer is the only one.

## `StaticObjectShift` — where the body must be for the feet it has

**Contract** — from the planted feet, compute the vertical shift the body needs, and start
the shift animation moving toward it at a fixed speed.

**Invariants** — up and down are decided by different rules, and this asymmetry is the
heart of the whole system:

- **upward** is the *average* of what the planted feet want. A foot on a step wants the
  body up; averaging shares the correction between the feet, which is what a person
  actually does.
- **downward** is the *maximum* over the planted feet of how far each leg can still reach
  — a limit, not a request. The body may not drop further than the shortest leg can
  follow, or the foot would leave the ground.
- when the downward limit itself demands a drop, it wins outright.

```text
FUNCTION static_object_shift(limb_states) -> real
  current = the shift animator's current value
  IF current is not a valid number THEN reset the animator ; current = 0

  up = average over planted feet of (that foot's collision correction + current),
       counting only positive contributions
  down = max over planted feet of that leg's reach limit at `current`

  IF down > 0            THEN shift = -down     # a leg demands the body drop
  ELSE IF -down < up     THEN shift = -down     # the reach limit binds before the average
  ELSE                        shift = up
  IF shift is not a valid number THEN reset the animator ; RETURN 0
  start the animator toward `shift` over |current - shift| / a fixed speed
  RETURN shift
```

**Notes** — the shift speed is a fifth of a metre per second, so the duration is
proportional to the distance and the *rate* is what is constant. That is the right shape:
a large correction takes proportionally longer and never snaps.

The two validity checks that reset the animator are not defensive padding. Degenerate
poses do occur — a character spawned inside geometry, a zero-length leg chain — and
without the reset the invalid shift persists and the character disappears below the world.

## `LegLengthShiftLimit`

**Contract** — over the planted feet, the largest downward shift any leg can still absorb.
Each limb answers for itself given the current shift; invalid answers are skipped.

## `PredictObjectShift` — anticipating the next footfall

**Contract** — over the feet currently in the air, find the one that lands soonest and
would need the body *lowered*, and aim the shift animator at that value over the time
until it lands. Failing that, if the body is currently held well below neutral, aim it
back at neutral over the time until the soonest landing. Reports whether it set anything.

**Invariants** — downward predictions win over upward ones unconditionally. Dropping the
body early is safe — the leg can always extend less — while raising it early lifts a
planted foot off the ground.

The upward correction is gated on the body being more than a threshold below neutral, so
the prediction does not fight small ordinary offsets.

```text
FUNCTION predict_object_shift(limb_states) -> bool
  FOR EACH foot in the air
    t = time until it plants ; s = the shift its landing will demand
    IF s < 0 AND t is the soonest so far THEN remember (t, s) as a downward prediction
    ELSE IF the body is more than the threshold below neutral AND t is soonest
         THEN remember t as an upward prediction
  IF a downward prediction exists THEN aim at its shift over its time
  ELSE IF an upward prediction exists THEN aim at neutral over its time
  ELSE RETURN false
  IF the time is essentially zero THEN use one frame instead   # never divide by zero
  RETURN true
```

**Notes** — the threshold below which the body is considered "held down" is three tenths
of a metre, a file-scope constant with no derivation. A second, commented-out constant
beside it suggests a correction magnitude was once applied as well.

## `Update`

**Contract** — the per-frame, non-pose half: advance the animation tracks, drop a freed
legs blend, record the current transform into the pose history, and let each limb update
its own footstep timing.

**Invariants** — the pose history is sampled here, on the frame, so that the prediction in
the pose callback has a movement history to extrapolate from. The two halves of this file
run at different rates and this is the only data that crosses between them.

## `PlayLegs`

**Contract** — adopt a blend as the one driving the legs. Everything that needs footstep
timing reads it.

**Notes** — in a debug build this also warns when an animation declares that it uses
footsteps but carries no footstep marks. That combination silently disables ground
collision for the limb, which is the single most common authoring error in this system.

## `LimbCalculate` / `LimbUpdate`

**Contract** — per limb: decide whether ground collision applies (only when the legs
animation carries footstep marks) and apply the limb's state; and per frame, let the limb
update itself against the object, the legs blend and the pose history.

## `Destroy`

**Contract** — unregister the pose callback and destroy every limb. The callback must go
first: a limb destroyed while the callback could still fire would be solved after it was
released.

## `Shift`

**Contract** — the character's current vertical offset. Read by anything that must agree
with where the body actually is — the camera especially, since a camera that ignores the
shift bobs against the body.
