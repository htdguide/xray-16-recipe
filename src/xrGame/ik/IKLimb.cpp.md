# src/xrGame/ik/IKLimb.cpp

> One leg's whole story for one frame: where the animation put the foot, where the ground
> actually is, how far the foot is allowed to move this frame, and the seven joint angles
> that get it there.

**Needs** — [`IKLimb.h`](IKLimb.h.md) · [`limb.h`](limb.h.md) · [`math3d.h`](math3d.h.md) · [`IKFoot.h`](../IKFoot.h.md) · [`ik_foot_collider.h`](../ik_foot_collider.h.md) · [`ik_anim_state.h`](../ik_anim_state.h.md) · [`ik_calculate_data.h`](../ik_calculate_data.h.md) · [`ik_calculate_state.h`](../ik_calculate_state.h.md) · [`ik_collide_data.h`](../ik_collide_data.h.md) · [`ik_limb_state.h`](../ik_limb_state.h.md) · [`ik_limb_state_predict.h`](../ik_limb_state_predict.h.md) · [`pose_extrapolation.h`](../pose_extrapolation.h.md) · [`GameObject.h`](../GameObject.h.md) · [`Kinematics.h`](../../Include/xrRender/Kinematics.h.md) · [`KinematicsAnimated.h`](../../Include/xrRender/KinematicsAnimated.h.md)
**Used by** — reached through its declarations in [`IKLimb.h`](IKLimb.h.md); callers name that, not this file.
**Tier floor** — T2. It is geometry and bookkeeping on data someone else owns; it allocates
nothing per frame, but nothing here needs explicit layout or deterministic destruction.
Its one T1-flavoured habit — reinterpreting the engine's matrix as the solver's matrix
because both are sixteen contiguous floats — exists only because two matrix types were
never reconciled, and a rebuild should convert instead.

## Purpose

This is the only file in the directory that knows what a foot is. Everything below it
solves an abstract seven-degree-of-freedom chain; everything above it schedules and
composes. This file decides, for one leg, **what the chain is being asked to reach** — and
that decision is almost all of the visible quality of the feature. The actual joint solve
is three lines near the end.

It exists as a separate unit from the per-object controller because there is one of these
per leg and their state is entirely independent: a limb's plant, its blend speed, its
memoized ground query and its predicted next footstep are its own. The only thing the
limbs must agree on is the body's vertical offset, and that is negotiated a level up.

## State

```text
RECORD Limb
  id            : int              # 0..3; also selects the default bone-name set
  bones         : list<BoneId>     # exactly 4: hip, knee, ankle, toe
  solver        : ChainSolver      # the 7-DOF solver, configured once at Create
  foot          : FootGeometry     # toe/heel/side points, foot normal, which bone is the reference
  skeleton      : SkeletonHandle
  collide       : bool             # whether this limb queries the world at all

  collider      : GroundQueryCache # memoized three-ray ground query
  collide_data  : GroundHit        # last frame's result: plane, contact point, did-it-hit
  anim_state    : AnimationFootState  # what the playing animation says: planted? glued? idle?
  saved_state   : LimbState        # last frame's accepted goal, blend speed, plant pose
  predicted     : StepPrediction   # time until the next footstep and the shift it will need
```

Invariants that are not obvious and are enforced by scattered assertions:

- `bones` is ordered root-outward and *contiguous in the skeleton*: bone `i+1` is a child
  of bone `i`. The solver's fixed `T` and `S` transforms are read from the bind pose of
  this chain at `Create` time and are wrong if the chain is not contiguous.
- Every goal matrix that reaches the solver is a rigid transform — determinant 1 within a
  tolerance of 0.2. The tolerance is that loose because the matrices are the product of a
  long chain of single-precision multiplications, not because any real scaling is allowed.
- `saved_state` is either *valid* (it holds the previous frame's accepted result and the
  frame it was computed on) or *invalid* (the limb has not run yet, or the animation was
  interrupted). Every blending decision checks this first, because blending against an
  absent previous frame is what produces a first-frame snap.

## `Create`

**Contract** — binds this limb to a skeleton: resolves four bone names to bone ids,
reads the foot geometry, and configures the chain solver from the skeleton's bind pose and
its authored joint limits. Runs once per object, at spawn. Allocates; never blocks.

```text
FUNCTION create(id, skeleton, collide_enabled)
  # Bone names: a built-in default per limb index, overridden per visual.
  # index 0,1 are the legs; 2,3 the arms, which no shipped creature enables.
  IF skeleton.user_data has section "ik"
    names <- skeleton.user_data["ik_limb<id>"].bones
  ELSE
    names <- default_bone_names[id]        # e.g. "thigh, calf, foot, toe" of the left leg
  bones <- resolve each name to a bone id

  # The two constant link transforms, straight out of the bind pose.
  T <- inverse(bind[hip])  * bind[knee]      # hip -> knee
  S <- inverse(bind[knee]) * bind[ankle]     # knee -> ankle

  limits <- authored_joint_limits(bones)     # see below
  solver.init(T, S, euler_convention = ZXY, euler_convention = ZXY,
              projection_axis = +Z, positive_axis = +X, limits)
```

**Notes**

*The joint limits are read from the model and then deliberately loosened.* Each authored
limit becomes `[π − high, π − low]` clamped to a full turn — the sign flip and offset
reconcile the model's convention with the solver's. Then: the hip's second and third
degrees are pulled *in* by one radian at the low end, the hip's first degree is capped at
`2π − 2π/3`, the knee is freed entirely, and all three ankle degrees are widened by a
radian at both ends. None of this is derivable from the model data; it is hand-tuning that
says *the authored limits are for ragdolls and are the wrong shape for a foot solve*. Two
consequences for a rebuild: the numbers are not load-bearing as numbers, and — because the
shipping path solves with limits switched off entirely — they are not load-bearing at all
unless the limit machinery is implemented.

*The knee is freed on purpose.* Its value comes from the law of cosines and has exactly one
admissible sign; constraining it further can only make a reachable goal unreachable.

## `Update`

**Contract** — called once per frame, *before* the pose is solved, for each limb. Refreshes
what the animation says about this foot, runs the ground query against the animated foot
pose, and predicts the next footstep. Performs ray queries against the collision database;
does not write any bone. Skips entirely if this limb does not collide or has no valid
previous state, and marks the ground result as "no hit" in that case so nothing downstream
uses stale geometry.

```text
FUNCTION update(object, legs_blend, pose_extrapolation)
  IF NOT collide OR NOT saved_state.valid
    collide_data.hit <- false
    RETURN

  anim_state.refresh(skeleton, legs_blend, id)   # planted? glued? idle? from motion marks
  anim_foot <- animated transform of the reference bone       # object space
  foot.ground_query(collide_data, collider, anim_foot, object.transform, object,
                    planted = anim_state.planted)
  step_predict(object, legs_blend, predicted, pose_extrapolation)
```

## `ApplyState` and `SetGoal`

**Contract** — the two entry points of the per-frame solve, called in this order by the
controller with one shared calculation record per limb. `ApplyState` settles which bone is
the contact reference for this frame and whether a ground query result may be used at all.
`SetGoal` produces this frame's goal matrix. Neither touches bone transforms.

```text
FUNCTION apply_state(cd)
  foot.choose_reference_bone()          # foot bone or toe bone, from the current pose
  cd.reference_bone <- foot.reference_bone
  cd.do_collide     <- collide AND cd.do_collide    # controller vetoes when the
                                                    # playing animation has no footstep marks
  cd.planted        <- anim_state.planted AND collide_data.hit

FUNCTION set_goal(cd)
  set_anim_goal(cd)                     # the uncorrected goal: animation x object transform
  hit <- none
  IF cd.do_collide
    collide_data.pick_dir <- straight down
    hit <- collide_data
  cd.ankle_to_toe <- transform from bone[2] to bone[3]   # needed by the foot fitter
  set_new_goal(hit, cd)
```

**Notes**

The veto in `apply_state` is the feature's on switch: **IK runs only while an animation
that carries footstep marks is playing.** Marks are the animator's statement of when each
foot is down. Without them there is no way to know whether a foot should be glued to the
ground or free to swing, and correcting a free foot to the surface under it produces a
creature that drags its toes. A rebuild needs the same per-animation metadata.

The pick direction is a constant "straight down". The file retains — disabled — a scheme
that steered the ray along the foot's own recent motion with a 1 % mix-in and a downward
bias, so that a foot swinging forward probed slightly forward. It was switched off, and
the reason is not recoverable from the source; the vector is still threaded through the
whole pipeline as if it varied.

## `SetNewGoal` — choosing what to reach for

**Contract** — the decision layer. Fits the animated foot to the ground it found, decides
whether this frame's goal may be taken immediately or must be approached over several
frames, and records the accepted result for next frame. Returns nothing; everything is
written into the shared calculation record and into `saved_state`.

**Invariants** — on exit, `cd.goal` is a rigid transform, `cd.blending` is true only if
`saved_state` was valid, and `saved_state` holds exactly what this frame accepted.

```text
FUNCTION set_new_goal(hit, cd)
  IF NOT cd.do_collide
    RETURN                                # goal stays as the animation posed it

  # How fast is the animation itself moving this foot? That sets the blend allowance.
  (cd.max_linear, cd.max_angular) <- difference(saved_state.anim_pos, cd.anim_pos)
  cd.max_linear <- cd.max_linear * 1.5

  # Fit the animated foot onto the surface: shift it along the pick direction until it
  # touches, and rotate it to lie in the plane. Reports whether the foot really is on
  # the ground; a foot the animation calls planted but that found no surface is not.
  cd.planted  <- foot.fit_to_ground(cd.goal, cd, hit, allow_rotation) AND cd.planted
  cd.blend_to <- cd.goal
  load saved_state into cd

  IF cd.planted
    set_new_step_goal(hit, cd)            # a planted foot is pinned, see below
  ELSE
    cd.blending <- saved_state.valid      # a free foot is approached, never snapped

  cd.correction <- cd.goal.position - cd.anim_pos.position   # what the body shift needs

  IF cd.blending
    blending(cd)

  saved_state <- cd
```

**Notes**

`cd.correction` is the only value this file exports upward. The controller averages the
vertical component over the planted legs to decide how far to raise the body.

## `SetNewStepGoal` — pinning a planted foot

**Contract** — runs when the foot is on the ground. Establishes or re-establishes the
*plant pose*: the fixed world transform the foot holds for as long as it is down. Chooses a
new plant when there was none, when the foot has just landed, when the animation is not
gluing this foot, or — for an idling creature — when the ground under the plant has moved.

```text
FUNCTION set_new_step_goal(hit, cd)
  IF NOT saved_state.valid OR NOT saved_state.planted OR NOT anim_state.glued
    cd.plant <- foot.fit_to_ground(cd, hit, allow_rotation)
    cd.unstuck_deadline <- now + random(500 ms, 1200 ms)
    reset blend speed to the animation's own foot speed
    # falls through: an idling limb may still re-plant below

  IF anim_state.auto_unstuck                 # the creature is standing, not walking
    cd.idle <- true
    fresh <- foot.fit_to_ground(cd, hit, allow_rotation)
    moved_a_lot   <- difference(cd.plant, fresh) exceeds 0.3 m or 45 degrees
    moved_at_all  <- difference(cd.plant, fresh) exceeds a hair
    IF moved_a_lot OR (now > cd.unstuck_deadline AND moved_at_all)
      cd.plant <- fresh
      cd.unstuck_deadline <- now + random(500 ms, 1200 ms)
      reset blend speed
      cd.blending <- true

  accelerate blend allowance
  IF cd.blending THEN cd.blend_to <- cd.plant ELSE cd.goal <- cd.plant
```

**Notes**

The plant is what stops foot slide. Without it, a body that keeps translating while a foot
is nominally down drags the foot across the ground, because the goal is recomputed from
the animation every frame and the animation is in the body's frame.

Two escape conditions, not one, and they are different failures. *Moved a lot* is the
surface genuinely changing — someone opened a door under the creature, or a physics prop
slid away — and must be followed immediately. *Moved at all, after a delay* is drift: the
accumulation of tiny disagreements that would otherwise leave a standing creature's foot
permanently a millimetre off the floor. The delay is randomized per re-plant so that a
squad standing together does not twitch in unison.

## `Blending` — rate limiting the correction

**Contract** — moves the accepted goal from where it was toward where it should be, by at
most this frame's allowance. Sets `blending` false the moment the allowance stops binding,
which is how "the correction has finished" is detected without a weight or a timer.

```text
FUNCTION blending(cd)
  IF saved_state.planted != cd.planted
    reset blend allowance to the animation's own foot speed   # the foot just changed regime

  candidate <- cd.blend_to
  reached   <- clamp_change(candidate, from = saved_state.goal,
                            max_linear = cd.max_linear, max_angular = cd.max_angular)
  cd.goal     <- candidate
  cd.blending <- NOT reached
  IF NOT cd.blending
    reset blend allowance

FUNCTION reset_blend_allowance(cd)
  IF the simulation is paused
    speeds <- 0                          # a paused frame must not accumulate correction
  ELSE
    speeds <- this frame's clamped change / frame duration

FUNCTION accelerate_blend_allowance(cd)
  IF NOT cd.blending
    reset_blend_allowance(cd); RETURN
  linear_speed  <- linear_speed  + 10 * frame_duration     # metres per second squared
  angular_speed <- angular_speed + 40 * frame_duration     # radians per second squared
  cd.max_linear  <- linear_speed  * frame_duration
  cd.max_angular <- angular_speed * frame_duration
```

**Notes**

`clamp_change` returns "the change was small enough that no clamping was needed", with
tolerances of 1e-7 on position and 5e-5 on rotation. Those tolerances, not a frame count,
are the definition of *arrived*.

The allowance **starts at the speed the animation is already moving this foot** and
accelerates from there. That is the reason the correction is invisible: it is never faster
than motion the eye has already accepted, and a foot that the animation is holding still
gets a near-zero allowance and therefore tracks the ground exactly.

Two alternative blend paths exist in the file and are both switched off by default: one
that blends in the *local* frame (composing the difference matrix toward identity rather
than moving the goal directly), and one that re-fits the blended pose to the ground on
every intermediate frame so a foot sliding toward its target never passes through the
floor. The second is obviously more correct and obviously more expensive; which way that
trade fell is not recoverable, but the code to enable either is intact.

## `Solve` and `CalculateBones`

**Contract** — the actual inverse kinematics. Converts the goal into the hip's frame, picks
the swivel angle from the animated knee, asks the chain solver for seven joint angles, and
re-evaluates the chain's bone transforms from those angles. Does nothing if the solver
reports the goal infeasible. Writes three bone transforms and everything below them in the
skeleton.

```text
FUNCTION solve(cd)
  IF NOT solver.set_goal(goal_in_hip_frame(cd), limits_on = false)
    RETURN                                     # unreachable: leave the animated pose alone

  knee <- desired_knee_position(cd)            # see below
  express knee in the hip's frame
  IF solver.solve_for_knee_position(knee, out angles)
    cd.angles <- angles
    calculate_bones(cd)

FUNCTION calculate_bones(cd)
  # Re-run the skeleton's own pose evaluation for this chain, but intercept the three
  # driven bones and substitute the solved rotations instead of the animated ones.
  # Everything hanging off the chain (the toe, anything attached to the foot) is then
  # recomputed correctly for free.
  override bone[0] with euler(angles[0..2]) about the bind frame
  override bone[1] with a single rotation of angles[3] about Y
  override bone[2] with euler(angles[4..6]) about the bind frame
  evaluate the sub-tree rooted at bone[0]
  remove the overrides
```

**Invariants** — the overrides are installed and removed within one call; any pre-existing
override on those bones is saved and restored, because other systems (a wound reaction, a
look-at) may own the same bones in the same frame.

**Notes**

The euler triples are applied as *negated* rotations about Z, then X, then Y, in that
order, starting from the bone's bind frame in its parent. The negation and the ordering are
the same convention mismatch the matrices carry; a rebuild that writes its own solver picks
one convention and none of this exists.

## `GetKnee` — inheriting the swivel from the animation

**Contract** — produces the knee position the solve should aim for: the animated knee,
carried rigidly onto the new hip→foot direction. Pure; reads the current pose only.

```text
FUNCTION desired_knee_position(cd) -> point
  hip        <- animated hip position
  foot       <- animated ankle position
  goal       <- corrected foot position
  old_dir    <- foot - hip
  new_dir    <- goal - hip
  IF |old_dir| is zero THEN RETURN animated knee    # degenerate pose, do not touch it

  # Split the hip->knee vector into a component along the old leg axis and one across it.
  along  <- component of (knee - hip) parallel to old_dir
  across <- the remainder
  # Re-attach: the parallel part follows the new axis, scaled so it stays the same
  # fraction of the way down the leg; the perpendicular part is carried unchanged.
  RETURN hip + new_dir * (along / |old_dir|^2) + across
```

**Notes**

This is the single most important line of art direction in the file. The swivel angle is
the solver's one free parameter, and the choice made here is *do not exercise it* — keep
the knee where the animator aimed it, relative to the leg. The alternative the solver
offers (pick the swivel that sits furthest inside the joint limits) produces technically
valid poses in which the knee rotates around the leg axis as the ground changes, which
reads as a broken hip.

## `ObjShiftDown` — the leg-length limit on the body shift

**Contract** — how far this leg would let the body drop before it has to stretch past its
own length. Positive means the body must come *up* by that much; the controller takes the
maximum over the planted legs and lets it override everything else. Pure.

```text
FUNCTION shift_down_limit(current_shift, cd) -> real
  hip   <- world position of the hip, with the current body shift already removed
  g     <- corrected foot position - hip
  L     <- total leg length (both link lengths)
  # The deepest the hip can be above the foot while still reaching it horizontally:
  reach <- sqrt(max(0, L^2 - g.x^2 - g.z^2))
  RETURN -g.y - reach
```

**Notes**

The horizontal offset between hip and foot is spent first; whatever length is left is the
vertical reach. Clamping the radicand at zero is not defensive coding — a foot further away
horizontally than the leg is long is a real situation during a wide stride, and it means
the leg cannot reach at all, so the limit degenerates to "the hip must be at the foot's
height".

## `step_predict` and `foot_matrix_predict`

**Contract** — looks ahead in the currently playing animation to the next footstep mark for
this limb, predicts the object's transform at that moment, runs the ground query against
the predicted foot pose, and reports how far the body will have to be shifted when the foot
lands, plus how long there is to get there. Costly — it re-evaluates the animation blends
at a future time and issues three more rays — and is the reason the prediction is only run
for feet that are currently in the air.

```text
FUNCTION step_predict(object, legs_blend, out state, pose_extrapolation)
  state.time_to_step <- time until this limb's next footstep mark, or infinity
  IF infinite THEN RETURN

  future_object <- pose_extrapolation.at(now + state.time_to_step)
  (foot, toe, reference) <- foot_pose_at(now + state.time_to_step)
  hit <- foot.ground_query(reference pose placed by future_object, planted = true)
  landed <- foot.fit_to_ground(reference pose placed by future_object, hit)
  state.shift <- landed.height - unfitted.height

FUNCTION foot_pose_at(time)
  # Advance a *copy* of every animation blend on the legs partition to `time`, read the
  # foot and toe bone transforms, then put the blends back exactly as they were.
  save all blends of the legs partition
  advance each blend by `time`; a blend that would have ended contributes nothing
  read the ankle and toe transforms out of a pose evaluation
  restore the saved blends
  RETURN the two transforms and which of them should be the contact reference
```

**Notes**

The save/advance/read/restore dance is the honest shape of the requirement: *evaluate the
animation at a future time without disturbing the present*. A rebuild whose animation
sampler is a pure function of (blend set, time) — which it should be — gets this for free
and deletes the whole routine.

A blend that would have finished before the target time has its contribution zeroed rather
than being removed, which quietly means the prediction is evaluated against a slightly
different blend mix than the one that will actually be playing. For a prediction horizon of
a fraction of a second this is below the noise floor.

## `GetHipInvert`, `Goal`, `transform`, `SwivelAngle`

**Contract** — frame plumbing, all pure reads of the current pose.

- `GetHipInvert` builds the inverse of the hip's *bind orientation placed at the hip's
  current position*. Goals are expressed in this frame: orientation from the bind pose,
  origin from the live pose. Using the live orientation instead would make the goal depend
  on the answer.
- `Goal` composes a world- or object-space matrix into that frame and converts it to the
  solver's matrix convention.
- `transform(bone_a, bone_b)` is the relative transform between two bones of this chain.
- `SwivelAngle` converts the currently animated knee and ankle positions into a swivel
  angle directly. It is the read-only counterpart of `GetKnee` and is not on the solve
  path.

## Notes on the file as a whole

*The matrix conversion.* One fixed permutation matrix maps the engine's axes to the
solver's, and matrices cross the boundary as `P · M` (for a transform whose frame is being
reinterpreted) or `P · M · P` (for a transform being conjugated into the other convention).
The engine matrix and the solver matrix are both sixteen floats in the same order, so the
original reinterprets the storage in place rather than copying. That aliasing is incidental;
the two conversions are not, and a rebuild that keeps a vendored solver must reproduce them
exactly or the legs bend sideways.

*Copy semantics.* A limb is copyable (the controller holds them in a growable array and
they move when it grows) but not assignable, and the copy has to re-point the saved state
at its new owner. This is an artifact of storing limbs by value in a resizable container;
a rebuild should either reserve the exact count up front — which the controller in fact
already does — or hold them by reference.

*Arms.* The default bone-name table has four entries: both legs and both arms. Nothing
ships that enables the arm limbs. The solver is general enough for them, and the goal
selection in this file — ground planes, footstep marks, plant poses — is not.
