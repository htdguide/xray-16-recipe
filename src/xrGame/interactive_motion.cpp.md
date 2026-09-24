# src/xrGame/interactive_motion.cpp

> Plays an authored death animation on a physically simulated body, and gives up into ragdoll the instant the animation stops being possible.

**Needs** — [`interactive_motion.h`](interactive_motion.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`interactive_motion.h`](interactive_motion.h.md)
**Tier floor** — T2: drives a physics body from an animation and hands it back

## Purpose

A ragdoll death is always physically plausible and almost always unconvincing. An authored
death animation is convincing and frequently impossible — the body was standing against a
wall, or on a slope, or half inside a doorway, and the animation would drive it through
geometry.

This file resolves that by playing the authored animation *through the physics body* rather
than instead of it, and treating any sign that the animation is failing as a cue to let go.
The result is a death that looks authored until the moment it cannot, and ragdoll from then
on. The hand-off must be seamless: the body's position and velocity at the moment of release
are whatever the animation left them at.

## State

`Stateless` — it operates the record declared in
[`interactive_motion.h`](interactive_motion.h.md).

## `setup`

**Contract** — binds a motion, a body and a heading angle, and arms the mechanism. Two forms;
the named one resolves the name through the body's skeleton first.

**Invariants** — the motion must be **non-looping**. A cyclic animation never reaches an end
and therefore never fires the abort callback, so the body would be driven by the animation
forever and never become a ragdoll. This is checked in debug builds and is the single most
important precondition in the file.

## `play`

**Contract** — starts the animation as a full-body cycle with the abort callback installed at
its end and this motion as the callback's payload, then runs the start hook.

**Invariants** — the callback is what converts "the animation finished normally" into the same
signal as "the animation failed". Both mean the same thing to the update loop: stop driving,
let go. A rebuild that distinguishes them will need a second path to release the body at a
normal end.

## `update`

**Contract** — one frame of the mechanism. The ordering is the whole of it:

```text
FUNCTION update()
  collide()                               # subclass: may set the abort flag
  IF NOT aborted THEN
    move_update()                         # subclass: may set the abort flag
    IF aborted THEN switch_to_free()      # release THIS frame, not next
```

**Invariants** — collision is tested **before** the move, so a body that is already
interpenetrating is released without being driven further into the geometry. And the release
happens in the same frame the abort is raised, so there is never a frame in which the body is
neither animated nor simulated.

An abort raised by the collision test skips the move entirely — the body is released on the
*next* update rather than this one, because the release is inside the not-aborted branch.
That one-frame asymmetry between the two abort sources is in the original; a rebuild that
releases immediately in both cases will differ by a frame at the moment a body strikes a wall.

## `switch_to_free`

**Contract** — hands the body back to ordinary ragdoll simulation, leaving it exactly where
the animation left it.

```text
FUNCTION switch_to_free()
  state_end()                              # disarm every flag
  owner = the object owning the body's first element
  owner.transform = the body's interpolated world transform
  skeleton.invalidate_bone_cache()
  skeleton.calculate_bones(forced)
```

**Invariants** — the order is load-bearing. The owner's transform must be taken from the
*interpolated* body transform, not the last simulation step's, or the release jumps by up to
one physics step. The bone cache must then be invalidated **and** recalculated immediately —
not deferred to the next frame's normal pose evaluation — because the ragdoll's first
simulation step reads the bone positions to seed itself, and a stale cache seeds it from the
animation's previous frame.

The interpolated transform is taken from the first element in the body's storage order, which
by construction is the root. That is a convention the physics body maintains, not something
this file establishes.

## `state_start` and `state_end`

**Contract** — the base start marks the mechanism started; the base end clears the abort flag,
disarms the mechanism and clears the started flag. Subclasses extend both to configure the
body — see [`imotion_velocity.cpp`](imotion_velocity.cpp.md).

**Invariants** — the end clears the *arming* flag as well as the started flag, so a motion that
has ended cannot be restarted without a fresh setup. That is intentional: the body has been
handed to the ragdoll and its state no longer matches what the animation expects.

## `destroy`

**Contract** — runs the end hook **only if the start hook ran**, then clears every flag.
Safe on a motion that was set up and never played, and on one that already ended.

**Invariants** — this is why the started flag exists. Running the end hook on a motion that
never started would configure a body the mechanism never took control of.

## `anim_callback`

**Contract** — the animation's end callback: sets the abort flag on the motion carried as the
callback's payload.

**Notes** — the payload is the motion object itself, recovered by an unchecked cast. A rebuild
with a typed callback channel avoids the cast; what must survive is that the animation system
can signal this specific motion without knowing what it is.

## `shell_setup`

**Contract** — in the shipped build, only an assertion that the body and its skeleton exist.
The source names it a hack for bipeds and it has been emptied out.
