# src/xrPhysics/PHElement.cpp

> One rigid body: how its mass is assembled from authored shapes, how its velocity is bled and clamped every step, and — the load-bearing part — how it takes ownership of a skeleton bone's transform and hands it back.

**Needs** — [`PHElement.h`](PHElement.h.md) · [`PHElementInline.h`](PHElementInline.h.md) · [`PHShell.h`](PHShell.h.md) · [`PHGeometryOwner.h`](PHGeometryOwner.h.md) · [`PHInterpolation.h`](PHInterpolation.h.md) · [`PHFracture.h`](PHFracture.h.md) · [`PHDynamicData.h`](PHDynamicData.h.md) · [`PHContactBodyEffector.h`](PHContactBodyEffector.h.md) · [`PHDisabling.h`](PHDisabling.h.md) · [`Physics.h`](Physics.h.md) · [`MathUtils.h`](MathUtils.h.md) · [`matrix_utils.h`](matrix_utils.h.md) · [`IPhysicsShellHolder.h`](IPhysicsShellHolder.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`PHElement.h`](PHElement.h.md)
**Tier floor** — T1: composes mass tensors in the library's layout and drives a body's integration directly.

## Purpose

Three separable jobs live here, and only the third is hard.

**Mass assembly** — turning authored bone shapes with authored masses into one body's mass, inertia
tensor and centre of mass, including the bookkeeping that lets a breakable split that mass later.

**Per-step conditioning** — the post-solve pass that scales, clamps, damps and sleeps a body. This
is where the simulation is kept from exploding, and every step of it is a defence against a
specific failure mode.

**The animation ownership contract** — who owns a bone's transform at which moment. This is the
seam the chapter is about, and getting it wrong produces either bodies that snap back to their
animated pose every frame or animated characters whose limbs drift away.

## State

```text
RECORD Element
  body            : handle to the simulated rigid body
  mass            : mass distribution (total, centre, inertia tensor)
  shell           : the owning Shell
  parent_element  : optional<Element>          # the bone hierarchy's parent, if any
  interpolation   : Interpolation              # the two-sample render bridge
  fractures       : optional<FracturesHolder>  # present only for breakables
  local_frame     : transform                  # this element's placement, in the shell's frame
  bone_id         : int                        # which skeleton bone this element owns

  linear_limit, angular_limit    : real        # velocity ceilings
  linear_scale, angular_scale    : real        # per-step velocity divisors (the ~1% bleed)
  air_linear, air_angular        : real        # resistance coefficients

  flags : set of {active, activating, needs_render_update, was_enabled_before_freeze,
                  enabled_on_step, fixed, animated}

  # invariant: `activating` means the element is active but has not yet been placed from its
  #            bone; the FIRST bone callback after activation performs the placement and clears it
  # invariant: `fixed` implies every force, torque and velocity write is ignored
  # invariant: the body's origin IS the centre of mass; the element's visible frame is the body's
  #            frame shifted back by `local_mass_center` (see PHElementInline.h)
```

## The animation ownership contract

**This is the part to read twice.** A skeleton bone's transform is written by one of two authorities
each frame: the animation system, or a physics body. The handover is done by installing a *callback*
on the bone; the presence of that callback is what transfers ownership.

```text
# Ownership states of one bone
#
#   no callback                 → animation owns the bone
#   callback = this element     → physics owns it; the element writes the bone each frame
#   callback = none, parameter
#     = an ancestor element     → the bone rides its ancestor's body; animation still poses it
#                                 locally, but the ancestor supplies the parent frame
```

**`set_bone_callback` / `clear_bone_callback`** install and remove that callback. A shell installs
one on every bone it has an element for, and a *reference* (callback absent, parameter present) on
every other visible bone so that lookups by bone find the nearest physics ancestor.

**`bones_callback`** is what runs when the skeleton evaluates that bone:

```text
FUNCTION bones_callback(INOUT bone)
  REQUIRE this element is active
  IF activating THEN
    # FIRST evaluation after activation: adopt the animated pose instead of imposing one
    activating_pos(bone.transform)
    bone.callback_overwrites_animation := true
  # normal case: overwrite the bone with this body's solved placement, expressed
  # relative to the shell's root
  bone.transform := shell_frame⁻¹ · this_element_frame
```

**`activating_pos`** is the handover in the other direction — physics adopting animation:

```text
FUNCTION activating_pos(bone_transform)
  to_bone_pos(bone_transform)        # teleport the body onto the animated bone, reset interpolation
  activating := false
  IF this element has no parent element THEN
    # the root element additionally records where the OBJECT's origin sits relative to this bone,
    # so the whole object can be placed from the root body afterwards
    shell.set_object_vs_shell_transform(bone_transform)
```

**Why an overwrite flag exists at all.** Skeleton evaluation is hierarchical: a bone's world
transform is its parent's times its local pose. A physics-owned bone's transform is absolute and
must not be composed with its parent again. The flag tells the skeleton evaluator "this bone's
transform is final". Without it a ragdoll's limbs compound their parents' rotations and the
character inflates.

**Why the first callback is special.** Activation happens at an arbitrary point in the frame, and
the animation system has the authoritative pose at that moment — not the caller who activated the
shell. So the element does not place itself at activation; it places itself the *first time the
skeleton asks it for a transform*, which is the first moment the animated pose is known to be
current. Every ragdoll in the game depends on this: a character dropping into a ragdoll must start
from the exact pose the death animation reached, or the transition pops.

**`static_root_bones_callback`** is the variant for a shell whose root does not move; it is
asserted-unreachable in the shipped build. Its body is preserved and does the same work plus
recording the object-in-root frame inline. Treat it as dead; the live path is
`bones_callback` plus `activating_pos`.

## `anim_to_vel` — driving a body from animation

**Contract** — sets the body's linear and angular velocity to whatever would carry it from its
current placement to the *animated* pose of its bone in `dt` seconds. Returns whether both required
velocities were within the given limits — that is, whether the body is actually keeping up.

```text
FUNCTION anim_to_vel(dt, linear_limit, angular_limit) -> bool
  target := object_frame · animated_pose_of(bone_id)
  current := body's current global frame
  difference := current⁻¹ · target
  dt := MAX(dt, tiny)

  target_mass_centre := mass centre under `target`
  linear  := (target_mass_centre - body.position) / dt
  angular := rotation_of(difference) expressed as an angular velocity over dt

  within := |angular| < angular_limit AND |linear| < linear_limit
  clamp both to their limits
  body.linear_velocity  := linear
  body.angular_velocity := angular
  RETURN within
```

**Notes** — this is the *blend rule* between animation and physics: rather than teleporting a body
onto its animated pose (which would deliver infinite force to anything it touches), the body is
given the velocity that will get it there, and the solver resolves the conflict with the world.
A limb that cannot reach its animated pose because something is in the way simply lags, and the
return value tells the caller the blend has failed — which is how an animated-physics character
decides to give up and go fully ragdoll.

`put_in_range` — clamp a vector's magnitude, reporting whether it clamped — is the shared helper.

## `ph_data_update` — the post-solve conditioning pass

**Contract** — runs once per step per element, after the solve. Reads and rewrites the body's
velocity, may disable the body, pushes an interpolation sample, and applies air resistance. Returns
early in three places, and each early return is a decision.

```text
FUNCTION ph_data_update(step)
  IF NOT active THEN RETURN
  IF fixed THEN zero velocity, force and torque; RETURN   # a fixed body accumulates nothing

  enabled_on_step := body.is_awake
  IF NOT enabled_on_step THEN RETURN                       # a sleeping body costs nothing

  # 1. the permanent bleed — divide, do not subtract
  body.linear_velocity  := body.linear_velocity  / linear_scale
  body.angular_velocity := body.angular_velocity / angular_scale

  # 2. clamp, and if clamping happened, re-measure
  IF |linear_velocity| > linear_limit THEN cut_velocity(linear_limit, angular_limit)
  IF |angular_velocity| > angular_limit THEN cut_velocity(linear_limit, angular_limit)

  # 3. sleep detection
  IF body.is_awake THEN run_sleep_accumulator()

  # 4. render bridge
  push a new interpolation sample
  IF body fell asleep during (3) THEN RETURN

  # 5. air resistance — angular is linear in speed, linear is QUADRATIC
  IF air_angular ≠ 0 THEN apply torque −angular_velocity · air_angular
  drag := |linear_velocity| · air_linear
  drag := MIN(drag, mass / fixed_step)                     # never reverse the velocity
  IF drag ≠ 0 THEN apply force −linear_velocity · drag
```

**Notes, in order of importance:**

*Linear drag is quadratic.* The coefficient multiplies speed to give a force-per-velocity, so the
applied force goes as speed squared. That is the correct model for a blunt object in air and is why
a falling crate reaches a terminal velocity rather than accelerating forever.

*The drag ceiling is a stability bound, not physics.* A drag force large enough to reverse the
velocity in one step makes the object oscillate and then explode. Clamping the coefficient at
`mass / fixed_step` makes the worst case "velocity goes to zero this step" — critically damped at
the step scale. Any rebuild applying velocity-proportional forces at a fixed step needs the same
bound.

*Clamping is not a simple scale.* See `cut_velocity` below.

*The `enabled_on_step` flag is remembered* because later phases and the breakable machinery need to
know whether this body participated in the solve, and by the time they ask, the body's sleep state
may have changed.

## `cut_velocity`

**Contract** — reduces a body's velocity to given limits *while keeping its placement consistent
with the velocity it actually had*. Not a simple scale.

```text
FUNCTION cut_velocity(linear_limit, angular_limit)
  (clamped_lin, did_lin) := limit(body.linear_velocity,  linear_limit)
  (clamped_ang, did_ang) := limit(body.angular_velocity, angular_limit)
  IF NOT (did_lin OR did_ang) THEN RETURN

  # advance the body by the DIFFERENCE, then adopt the clamped velocity
  body.linear_velocity  := clamped_lin  - body.linear_velocity
  body.angular_velocity := clamped_ang  - body.angular_velocity
  integrate_body_one_step(body, fixed_step)
  body.linear_velocity  := clamped_lin
  body.angular_velocity := clamped_ang
```

**Notes** — this is subtle and worth stating plainly. The solver has already integrated the body
forward using the *unclamped* velocity. Simply overwriting the velocity would leave the body one
step further along than its new velocity can account for, which shows up as a jump. So the body is
integrated *backwards* by the excess (the difference is negative when clamping down), putting it
where it would have been had the clamp applied during the solve. A rebuild that clamps before the
integration instead of after does not need this, and should prefer that.

## `ph_tune`

**Contract** — pre-solve. Applies any contact effector that accumulated on this body during the
collision phase (the merged liquid/slow-down drag from [`Physics.cpp`](Physics.cpp.md)) and
asserts the body is still inside the world.

## Mass assembly

**`add_mass(shape, offset, mass_centre, mass, fracture)`** — folds one authored bone shape's mass
into this element's, *recomputing the element's mass centre* and translating the existing tensor
to the new centre.

```text
FUNCTION add_mass(shape, offset, shape_mass_centre, mass, optional fracture)
  m := mass distribution of `shape` scaled to `mass`     # box, sphere or cylinder analytic form
  rotate m by the shape's own orientation, translate to the shape's centre
  rotate m by `offset`                                   # fold the bone's frame in

  new_centre := (this.centre · this.mass + offset·shape_mass_centre · mass) / (this.mass + mass)
  translate m            to be about new_centre
  translate this.mass    to be about new_centre
  IF this element has fractures THEN distribute the added mass across every fracture's parts
  IF a fracture was named THEN credit the added mass to that fracture's SECOND part
  ASSERT the result is a valid mass distribution
  this.mass := this.mass + m
  this.centre := new_centre
```

**Invariants** — the whole scheme rests on *both tensors always being expressed about the same
point*. Every translate in that sequence exists to maintain it, and the mass subtraction breakables
depend on (`mass_sub` in [`Physics.cpp`](Physics.cpp.md)) is only correct while it holds.

The cylinder case builds an orthonormal basis from the cylinder's axis before rotating — the
authored data gives a direction, not a frame, so the other two axes are arbitrary and generated.
That is sound because a cylinder's inertia tensor is symmetric about its axis.

**`calculate_it_data_use_density`** — the other path: sum every shape's own analytic mass at a
uniform density, about a given centre. Used for objects with no authored per-bone mass.

**`recursive_mass_summ`** — the breakable variant. Walks the element's shapes in fracture order,
accumulating each fracture's *first part* mass as it goes and recursing to compute the *second
part*. Each fracture therefore ends up knowing the mass on each side of the break before anything
breaks, which is what the break criterion needs.

**`re_adjust_mass_positions`** — relocate every shape by a pivot shift and recompute the mass,
choosing the authored-per-bone path when a skeleton is available and the uniform-density path
otherwise. This is how a fragment that separates from a breakable gets a sensible mass.

**`smooth`-adjacent note** — `set_mass` distributes a requested total across shapes by volume;
`set_mass1` divides it equally. Both are on the shell, not here; see
[`PHShell.cpp`](PHShell.cpp.md).

## Impulses

**Contract** — four flavours, distinguished by what the position argument means and what happens
afterwards. All of them wake the body, all of them refuse on a fixed body, and all of them clamp
through `BodyCutForce`.

```text
FUNCTION apply_impulse_vs_mass_center(offset_from_centre, direction, magnitude)
  force := direction · (magnitude / fixed_step)         # an impulse is a one-step force
  apply force at a point given RELATIVE to the body's centre, in the body's frame
  clamp force and torque

FUNCTION apply_impulse_vs_global_frame(world_point, direction, magnitude)
  the same, with the point given in world space

FUNCTION apply_impulse_trace(world_point, direction, magnitude, bone_id)
  # a hit attributed to a specific bone, which may not be this element's own bone
  IF bone_id = this element's bone THEN offset := world_point − mass_centre
  ELSE IF a skeleton exists THEN
    offset := (this_bone_transform⁻¹ · that_bone_transform) applied to world_point, minus mass_centre
  ELSE offset := 0                                      # no skeleton: treat as a central hit
  apply_impulse_vs_mass_center(offset, direction, magnitude)
  IF breakable THEN record the impulse against the fracture whose shapes cover that bone
```

**Notes** — an impulse is expressed as a force applied for exactly one fixed step, which is why
every one of them divides by `fixed_step`. That makes an impulse's effect *independent of the step
size*, which the determinism requirement needs; a rebuild that applies impulses as instantaneous
velocity changes gets the same result more directly and should.

The bone-attribution path exists so that shooting a breakable object's leg breaks the leg off,
rather than nudging the whole object. The impulse goes to the body either way; what the bone
identity buys is the entry in the fracture's impact list.

`apply_impact` is the replay form: an impulse recorded earlier, reapplied to a newly-created
fragment so the fragment flies off in the direction of the blow that separated it.

## Placement

**`set_transform`** — place the body so that the element's *visible* frame equals the given one.
The body's own origin is the mass centre, so the requested frame is converted first. Resets the
sleep accumulator, marks the render bridge dirty, re-indexes the shell spatially, and optionally
clears the shapes' motion history (a teleport must not be swept).

**`transform_position`** — apply a transform *to* the body's current placement rather than
replacing it. Resets both interpolation samples, because the motion this implies is not real.

**`interpolate_global_transform`** — the render query. If nothing has changed since the last call
it returns the solved placement directly; otherwise it blends the two interpolation samples and
shifts back from the mass centre to the element's visible frame.

**`get_global_transform_dynamic`** — the same without blending: the true current state. Used by
anything that must reason about the simulation rather than draw it — joint construction, breakable
splitting, network export.

The mass-centre shift is the small inline pair in
[`PHElementInline.h`](PHElementInline.h.md), and getting its direction backwards is the classic way
to make every object in the game render offset from its collision.

## Lifecycle

```text
FUNCTION build()
  body := create rigid body, asleep
  IF this element has NO shapes THEN fix()               # a shapeless element is an anchor
  ELSE  REQUIRE mass > 0 ; body.mass := this.mass
  body.position := mass_centre                           # body origin IS the mass centre
  reset the sleep accumulator
  build the shape composite and bind every shape to this body

FUNCTION run_simulation()
  add this element's shape group to the shell's collision space
  IF the body is not yet in an island THEN shell.island.add_body(body)
  wake the body

FUNCTION activate(frame, linear_velocity, angular_velocity, start_disabled)
  REQUIRE NOT active
  local_frame := frame
  build() ; run_simulation()
  set_transform(frame)
  body velocities := the given ones
  bind the interpolation to the body
  IF start_disabled THEN put the body to sleep
  active := true ; activating := true                    # placement happens at the first bone callback
  IF a skeleton exists THEN install the bone callback

FUNCTION deactivate()
  destroy the shapes and the body, remove it from the island
  active := false ; activating := false
  IF this element owns its bone's callback THEN release it
```

**Notes** — the `activate` overload taking two transforms and a blend fraction derives a linear
velocity from the *difference of their origins* and leaves angular velocity zero. The fraction
argument is ignored. It is used when a moving animated object becomes physical and needs to keep
its motion; the omission of angular velocity means a spinning object loses its spin at the
handover, which is visible on thrown objects and appears to be an oversight rather than a choice.

`preset_active` is the third activation path, used when a shell is assembled from parts of another
shell mid-step (breakable splitting): it places the element from its bone, records the object-in-root
frame if it is the root, and starts simulating — but skips the bone-callback installation, because
the caller is about to move the element to a different shell.

## `set_shell`

**Contract** — move this element's body from one shell's island to another's. Must be called
outside a step; both islands must be unmerged.

## `fix` / `release_fixed`

**Contract** — `fix` makes the body immovable: it stops updating its own position, gets an
enormous mass and inertia, loses gravity and has its state zeroed. `release_fixed` restores the
real mass and gravity. The pair exists because a door or a mounted object alternates between the
two, and the dynamics library's genuinely-static bodies cannot be un-fixed.

**Notes** — a *shapeless* element is fixed at build time and never released. That is how a shell
expresses "this bone is an anchor point": it has a joint but no geometry and no mass of its own.

## `split_process` / `pass_end_geoms`

**Contract** — the element's half of breakable splitting. `pass_end_geoms` moves a contiguous range
of shapes to another element, detaching them from this body and renumbering the survivors'
indices. `split_process` asks the fractures holder to produce the new elements and deletes the
holder when no fractures remain. The algorithm is in [`PHFracture.cpp`](PHFracture.cpp.md).

**Invariants** — shapes are moved as a *contiguous range*, and every fracture's recorded range is
adjusted for the shift. This is why the shell builder adds shapes in bone-hierarchy order: a
subtree of the skeleton is always a contiguous range of shapes, so a break is always a range
operation. A rebuild that stores shapes unordered must replace ranges with explicit sets and
rewrite the whole of [`PHShellSplitter.cpp`](PHShellSplitter.cpp.md).
