# src/xrGame/physics_shell_animated.cpp

> A rigid-body assembly that is driven by an animation rather than solved: it exists so that a skeletally animated thing can push and be pushed without the solver ever taking over its pose.

**Needs** — [`physics_shell_animated.h`](physics_shell_animated.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`physics_shell_animated.h`](physics_shell_animated.h.md)
**Tier floor** — T1: the physics seam's per-bone bodies, driven each frame from a pose

## Purpose

Some animated things must be *felt* by the physics world without being *run* by it: a
moving platform, a door, a creature whose animation must stay authoritative. This type is
the arrangement that makes that work — a full per-bone body assembly built from the model's
collision proxies, permanently flagged as animated, with its own collision response disabled,
whose bodies are teleported onto the animation's pose every frame.

The distinction from a ragdoll is the whole content of the file: a ragdoll is the same
assembly with the flags reversed.

## State

```text
RECORD AnimatedPhysicsShell
  physics_shell   : rigid-body assembly   # one body per collision-proxy bone
  update_velocity : bool                  # derive body velocities from the animation
```

**Invariants**
- The assembly is created in the constructor and destroyed with this object. It is never
  absent while this object lives.
- The assembly is permanently in *animated* mode and has collision response disabled. Both
  are set once at creation and never toggled; a rebuild that toggles them has built a
  ragdoll by accident.

## Construction

**Contract** — builds a rigid-body assembly for the holder's model from its collision
proxies, with no bone-parameter overrides, then puts it into the animation-driven
configuration.

```text
FUNCTION create_shell(holder)
  physics_shell = build assembly for holder's model   # every collision-proxy bone
  physics_shell.pose bodies from the model's current bone transforms
  physics_shell.disable collision response
  physics_shell.animated = true
```

**Notes** — **disabling collision response is not the same as disabling collision.** The
bodies still occupy space and still register contacts, so other bodies are pushed out of
them; what is suppressed is this assembly reacting to those contacts. That asymmetry is the
entire mechanism by which a moving door pushes a crate and is not itself budged.

Posing the bodies from the bone transforms at creation, before the first update, matters
because a body created at the origin and teleported on the next frame sweeps the whole level
in the intervening step and collects contacts with everything it passes through.

## `update`

**Contract** — drives the assembly from a world transform for one frame. Optionally derives
each body's linear and angular velocity from how far it moved, then installs the transform,
recomputes the skeleton's bone matrices, and teleports every body onto its bone.

```text
FUNCTION update(world_transform) -> bool
  IF update_velocity
    physics_shell.derive velocities from the pose change over the frame delta,
                         clamped at ten times the default linear and angular limits
  physics_shell.transform = world_transform
  physics_shell.kinematics.recompute bone matrices
  physics_shell.pose bodies from the bone transforms
  RETURN true
```

**Invariants** — the order is fixed and each step depends on the previous: velocities must
be derived from the *previous* pose before the new transform is installed, the bone matrices
must be recomputed before the bodies can be posed from them, and the bodies must be posed
after the transform so they land in world space rather than model space.

**Notes** — the velocity clamp is **ten times** the assembly's ordinary linear and angular
velocity limits. An animation can move a bone arbitrarily fast — a swung arm, a snapped
door — and the derived velocity is what other bodies are struck with. Clamping at the
ordinary limit would make animation-driven impacts feel weak; not clamping at all would let a
single fast animation frame launch everything it touches out of the level. Ten is a feel
value with no derivation in the source.

Deriving velocity is optional because it costs a pass over every body and only matters when
the animated thing must *impart* motion. A door that only blocks does not need it.

The frame delta used is the device's, not the owner's scheduled delta. An animated assembly
belonging to an object the scheduler is updating at a reduced rate therefore derives
velocities against the wrong interval. Unrecovered: whether that is deliberate — animated
assemblies in the shipped data belong to objects that update every frame, so it does not
arise.

## Destruction

**Contract** — releases the assembly through the physics seam's own destroy path, which
unregisters every body from the world before freeing it. This is the teardown-ordering rule
the engine asserts at runtime, satisfied at this one seam.
