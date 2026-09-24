# src/xrGame/interactive_animation.cpp

> An animation driving a physics body that fades itself out as soon as the body pushes into the world too far.

**Needs** — [`interactive_animation.h`](interactive_animation.h.md) · [`physics_shell_animated.h`](physics_shell_animated.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`xrPhysics/ExtendedGeom.h`](../xrPhysics/ExtendedGeom.h.md) · [`xrPhysics/MathUtils.h`](../xrPhysics/MathUtils.h.md) · [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`interactive_animation.h`](interactive_animation.h.md)
**Tier floor** — T2: reads contact depths out of the collision step

## Purpose

An animation that moves a body through space — a creature lunging, a corpse settling — will
sometimes drive it into a wall. This file detects that and responds by **halving the
animation's weight and stopping it**, so the pose falls back toward whatever else is playing
rather than continuing to intersect geometry.

The whole mechanism rests on one measurement that is not otherwise available: how deeply the
body is interpenetrating anything this frame. That number can only be read from inside the
collision step, which is why the file installs a contact filter whose real job is measurement
rather than filtering.

## State

```text
RECORD InteractiveAnimation
  blend : optional<Blend>   # the animation being watched; cleared when it stops

# module-scope, written by the contact filter and read by the collide test
deepest_penetration : real
```

**Invariants** — the penetration depth is module-scope rather than per-instance because the
contact filter is a plain function the physics library calls with no user payload. Only one
body may run its collision pass at a time, which is true because the collision pass is
synchronous — the depth is zeroed immediately before the pass and read immediately after. A
rebuild whose collision step is concurrent must carry the depth per body.

## `collide`

**Contract** — zeroes the recorded depth, runs the body's full collision pass, and answers
whether the deepest contact exceeded the tolerance.

```text
FUNCTION is_colliding() -> bool
  deepest_penetration = 0
  body.collide_all()          # the contact filter runs and records depths
  RETURN deepest_penetration > 0.05     # 5 cm
```

**Invariants** — the tolerance is five centimetres, not zero. A body in contact with the
ground is always penetrating slightly — that is how the solver expresses resting contact —
so a zero tolerance would abort every animation of a body standing on anything. Five
centimetres is deep enough to be a real intersection and shallow enough to catch a wall before
the body is visibly inside it.

**Notes** — the collision pass is run **for the measurement**, not for the simulation. Its
contacts are discarded. That is a real cost, and it is paid because there is no other way to
ask "how far into the world am I" of this physics interface.

## `contact_callback`

**Contract** — invoked once per contact during the pass. Records the maximum depth seen,
**except** for contacts between two parts of the same object.

```text
FUNCTION on_contact(contact)
  self, other = the two geometries' owners
  IF other exists AND other is the same object as self THEN RETURN   # self-contact
  deepest_penetration = max(deepest_penetration, contact.depth)
```

**Invariants** — the self-contact exclusion is essential: an animated skeleton's own limbs
touch each other constantly (arms against the torso, thighs against each other) and every one
of those is a deep penetration. Without the exclusion no animation of an articulated body
would ever survive a frame.

The filter does not actually *refuse* any contact — it only measures. The refusal path exists
in the source and is disabled.

## `update`

**Contract** — advances one frame and answers whether the animation is still driving the body.

```text
FUNCTION update(transform) -> bool
  IF no blend THEN RETURN false                # already finished
  IF the animated body's own update says stop THEN RETURN false

  IF the blend is playing AND is_colliding() THEN
    fire the blend's own end callback, if it has one
    blend.weight = blend.weight / 2            # fade, do not cut
    blend.playing = false

  IF the blend is not playing THEN
    forget the blend
    RETURN false
  RETURN true
```

**Invariants**:

- **The weight is halved rather than zeroed.** Cutting it to zero pops the pose; halving it
  leaves the animation contributing at reduced weight while the ordinary blend machinery
  fades it the rest of the way. That is the difference between a creature stopping against a
  wall and a creature teleporting into its idle pose.
- **The blend's own end callback is fired on the collision path.** Whatever was waiting for
  this animation to finish — a game-layer state change, a sound — gets its notification even
  though the animation was cut short. Skipping it strands the waiter.
- **The blend reference is cleared once it stops**, and a cleared reference makes every
  subsequent update answer false immediately. The mechanism is one-shot.

## `create_shell`

**Contract** — builds the body through the base and then installs the contact filter on it.
The filter must be attached at construction because the collision pass is invoked from inside
`collide` with no opportunity to pass one in.

## Construction

**Contract** — takes the owner and the blend to watch, and constructs the animated body with
the physics simulation initially *not* driving it: the animation drives, the physics
measures.
