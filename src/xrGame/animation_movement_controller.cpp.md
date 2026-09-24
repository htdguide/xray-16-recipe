# src/xrGame/animation_movement_controller.cpp

> Lets an animation drive the object's world transform: the root bone's displacement is stripped out of the pose and applied to the object instead.

**Needs** — [`animation_movement_controller.h`](animation_movement_controller.h.md) · [`poses_blending.h`](poses_blending.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`xrPhysics/matrix_utils.h`](../xrPhysics/matrix_utils.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: rigid-transform algebra and a bone callback; no byte layout, but it runs inside the per-frame pose evaluation and must not allocate there

## Purpose

Most of the time an object's position is decided by movement code and the animation only
poses the mesh around it. A class of authored motions inverts that: a death fall, a
climb, a scripted use animation, a vehicle mount. There the *animation* knows where the
body ends up, and any movement code fighting it produces sliding feet or a body that ends
the motion a metre from where the animator placed it.

This controller resolves the conflict by moving the displacement from one place to the
other. Every time the skeleton is evaluated it forces the root bone's local transform to
identity — so the pose is rendered *in place* — and separately samples what that root
transform would have been, composing it onto a captured base transform to produce the
object's world transform. The mesh no longer walks away from its own origin; the object
does.

It is a separate file because it is the one piece of the game layer that reaches into the
pose evaluation itself. Everything else treats a skeleton as a thing that is posed after
the object is placed.

## State

```text
RECORD animation_movement_controller
  object_transform  : reference to the owner's world transform  # written every frame, never copied
  start_transform   : matrix      # the world transform the motion is measured from
  poses_blend       : poses_blending   # smooths the jump onto the animation's path
  kinematics        : reference to the posed model
  animated          : reference to the same model's animation side
  control_blend     : optional<reference to a playing blend>   # none == inactive
  initial_blending  : bool
  stopped           : bool
  blend_linear_speed, blend_angular_speed : real
```

**Invariants**

- `control_blend` present is exactly the definition of *active*. While active, the root
  bone carries this controller's callback and the model carries this controller as its
  blend-destroy listener; when it goes absent both are gone. There is no half-installed
  state, which is why the destructor and the blend-destroy notification funnel into one
  teardown.
- While active, nobody but this controller may write `object_transform`. That is asserted
  every frame against a shadow copy in debug builds, because a movement system that keeps
  running underneath produces a drift that is otherwise very hard to attribute.
- The root bone must have no other callback when the controller is constructed. The bone
  callback slot is single-occupancy, so root motion and any other root-bone effect are
  mutually exclusive by construction.
- `blend_linear_speed` / `blend_angular_speed` are computed but not read on the shipped
  path — see Notes on the blending step.

## construction

**Contract** — takes the object's world transform (by reference, to be written in place),
the transform the motion starts from, the model and the blend that drives it. Installs the
root-bone callback, zeroes the root bone, registers for blend destruction, forces one full
bone evaluation so the first sample is valid, and arms the entry blending. Allocates
nothing. After it returns the object is under animation control.

The ordering is load-bearing: the callback must be installed *before* the forced
evaluation, or the first evaluated pose keeps its root displacement and the object jumps
by one frame of motion.

```text
FUNCTION construct(object_transform, initial_pose, model, blend)
  start_transform = initial_pose
  control_blend   = blend
  root = model.bone_instance(model.root_bone)
  REQUIRE root has no callback          # single-occupancy slot
  root.set_callback(custom, root_bone_callback, self)
  root.transform = identity
  measure_entry_speed()
  model.set_blend_destroy_listener(self)
  model.invalidate_bones(); model.evaluate_bones()
  arm_pose_blending()
```

## `OnFrame`

**Contract** — called once per frame while active, from the owner's update. Samples the
animation's root position, composes it onto the base transform, and either interpolates
the object toward that or snaps to it. Does not evaluate the skeleton itself; it relies on
the owner's normal pose evaluation having run.

```text
FUNCTION on_frame()
  REQUIRE active
  verify_nobody_else_moved_us()

  root_pos = animation_root_position()
  target   = start_transform COMPOSED WITH root_pos

  IF NOT poses_blend.target_reached(control_blend.time_current) THEN
    object_transform = poses_blend.pose(control_blend.time_current)   # still easing in
  ELSE
    object_transform = target
```

## root position sampling

**Contract** — answers "where would the root bone be if *only this blend* were playing, at
full weight". Pure with respect to the model: it saves and restores the blend's weight and
does not disturb the pose.

This is the subtle part of the whole file. The skeleton is usually driven by several
blends at once — an upper body aiming while the legs run — and the accumulated root
transform mixes all of them. Moving the object by that mixture would make the object's
path depend on whatever unrelated animation happened to be playing. So the sampler builds
the key table for the root channel, finds the key belonging to *this* blend, discards every
other blend and every other channel, forces the weight to one, and rebuilds a single bone
matrix from that lone key.

```text
FUNCTION animation_root_position() -> matrix
  keys = animated.build_dequantized_bone_keys(root_bone_data, channel 0)
  key  = the key in channel 0 whose blend is control_blend
  FAIL WITH "root motion blend vanished" IF key is none

  saved_weight = control_blend.weight
  control_blend.weight = 1
  keys = a table holding only (control_blend, key) in channel 0; all other channels empty

  bone = build_bone_matrix(from identity parent, keys)
  control_blend.weight = saved_weight
  RETURN bone.transform
```

**Notes** — the sample is taken from the *first* bone instance rather than the named root
bone. The two are the same in every shipped skeleton; a rebuild should sample the declared
root bone and lose nothing.

## `NewBlend`

**Contract** — hands control to a different blend without dropping out of animation
control, which is how an authored motion chain (fall, then settle, then die) keeps one
continuous object path. Takes the new blend, a transform to restart from, and a flag
saying whether the outgoing motion was *local* — meaning its root displacement is relative
to where the chain currently is, rather than absolute.

Three cases, and the difference between them is where the new base transform comes from:

```text
FUNCTION new_blend(blend, new_transform, local_animation)
  REQUIRE active
  keep_blending = NOT poses_blend.target_reached(control_blend.time_current)

  IF stopped THEN
    # the chain was explicitly broken; restart from the caller's transform
    start_transform = new_transform
    control_blend   = blend
    measure_entry_speed()
    keep_blending   = true
    stopped         = false

  ELSE IF local_animation THEN
    # fold the outgoing motion's FULL displacement into the base, so the new
    # motion starts where the old one finished rather than where it began
    saved = control_blend.time_current
    control_blend.time_current = control_blend.time_total - ONE_SAMPLE
    start_transform = start_transform COMPOSED WITH animation_root_position()
    control_blend.time_current = saved

  # else: the two motions share a base transform; nothing to accumulate

  control_blend = blend
  IF keep_blending THEN arm_pose_blending() ELSE poses_blend = inactive
```

**Notes** — the outgoing motion is sampled one sample period short of its end rather than
at its end, because the last sample is the boundary between this motion and whatever
follows and sampling exactly on it is ambiguous. A rebuild whose sampler clamps inclusively
can sample at the end directly.

**Notes** — if the entry blending from the previous motion had not finished, it is re-armed
for the new one rather than abandoned: an interrupted ease-in must not snap.

## entry blending

**Contract** — the object is somewhere; the animation says it should be somewhere slightly
else. Rather than teleport on the first frame, the controller interpolates the object from
where it was to where the animation puts it, over the first fifth of the motion.

```text
FUNCTION arm_pose_blending()
  blending_time = 0.2 * control_blend.time_total     # a fifth of the motion

  saved = control_blend.time_current
  control_blend.time_current = blending_time
  target = start_transform COMPOSED WITH animation_root_position()
  control_blend.time_current = saved

  poses_blend = blend_from(object_transform, target, over blending_time)
```

**Notes** — a *fraction* of the motion rather than a fixed duration, so a long slow motion
gets a long correction and a short violent one gets a short one. There is no derivation for
one fifth; it is the value that stopped being visible.

**Notes** — the file also measures the animation's linear and angular speed at the first
frame (by advancing the blend one frame delta and differencing the two root positions) as
input to an alternative, rate-limited correction. That correction is not the one on the
shipped path — the pose interpolation above replaced it — so the measured speeds are dead
weight. A rebuild should implement the pose interpolation and skip the measurement
entirely.

## root bone callback

**Contract** — invoked by the pose evaluation every time the root bone's matrix is built.
Forces it to identity. This is the mechanism by which the animation's displacement is
removed from the rendered pose; the object transform picks it up instead, and applying it
in both places would double the motion.

**Notes** — the callback is a raw hook into the renderer's pose evaluation with the
controller carried as an opaque parameter. What a rebuild owes here is not the hook but its
two properties: it runs inside pose evaluation (so it must be cheap and allocation-free)
and it runs for exactly one bone.

## `IsActive`, `IsBlending`, `stop`, `ObjStartXform`, `start_transform`, `ControlBlend`

**Contract** — `IsActive` is "a controlling blend is present". `IsBlending` is true only
while the controlling blend is still accruing weight, which callers use to know the object
is not yet fully committed to the animation. `stop` marks the chain broken, so the next
`NewBlend` restarts from the caller's transform instead of accumulating. The remaining
three are readers of the base transform and the controlling blend.

## teardown

**Contract** — removes the root-bone callback, deregisters the blend-destroy listener and
drops the blend, making the controller inactive. Idempotent in effect because it only runs
when active. Reached from two directions — the owner destroying the controller, and the
renderer destroying the blend that drove it — and both must leave the root bone's callback
slot free, or the next root-motion animation on this model cannot install itself.

**Notes** — the blend-destroy notification is the important half. The blend is owned by the
animation system and can end on its own schedule; the controller must learn about it rather
than poll, because between the blend's death and the next update the pointer would be
stale. A rebuild needs a notification, not necessarily a callback interface.
