# src/xrGame/ActorEffector.cpp

> The actor's camera- and screen-effect stack: how a data-driven shake or a full-screen filter is attached to the player, how strongly it applies, and how the first-person weapon model is kept partly exempt from it.

**Needs** — [`ActorEffector.h`](ActorEffector.h.md) · [`CameraEffector.h`](CameraEffector.h.md) · [`PostprocessAnimator.h`](PostprocessAnimator.h.md) · [`Actor.h`](Actor.h.md) · [`xrEngine/EffectorPP.h`](../xrEngine/EffectorPP.h.md) · [`xrEngine/ObjectAnimator.h`](../xrEngine/ObjectAnimator.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md)
**Used by** — [`ActorEffector.h`](ActorEffector.h.md)
**Tier floor** — T2: per-frame quaternion interpolation and matrix composition

## Purpose

An *effector* is something that modifies the camera or the screen for a while: an
explosion's shake, drunkenness, a psychic attack, the death fade, a scripted cutscene
move. The engine owns the stack and the blending
([`xrEngine/CameraManager.cpp`](../xrEngine/CameraManager.cpp.md)); this file owns the
**game's vocabulary on top of it**: effectors are described in configuration, so adding a
new screen effect is a data change, and the strength of an effect can be driven by a live
game value rather than only by elapsed time.

The second thing here is the **heads-up camera**, a second camera pose maintained beside
the real one. The first-person weapon model is drawn from it, and an effector declares
whether it affects it. That is how an explosion can shake the world violently while the
gun in your hands stays where your hands are — or shake both, when the data says so.

## State

```text
RECORD EffectorDescription            # not a record in the code: these are ltx keys
  pp_eff_name     : text      # the screen-effect animation file
  pp_eff_cyclic   : bool      # loops forever, or plays once and expires
  pp_eff_overlap  : bool      # default false; composes with other screen effects
  cam_eff_name    : text      # the camera-motion animation file
  cam_eff_cyclic  : bool
  cam_eff_hud_affect : bool   # optional; whether the weapon camera follows this one
```

**Invariant** — a section may define either half, both, or neither. Each half is installed
only if its name key is present, so one section describes a paired camera-and-screen
effect and another describes a screen tint with no motion.

**Invariant on the controller** — an effector controller holds back-pointers to the two
effectors it drives and asserts on destruction that both are already gone. Outliving your
driver is the one failure this system cannot tolerate: the effector would call into freed
memory every frame. A rebuild with a different ownership model still needs the same
guarantee — *the strength source must outlive the effector, and the effector must
un-register itself*.

## `AddEffector` — four strength models

**Contract** — installs the effectors a configuration section describes onto the actor,
tagged with an effect *type* that also serves as its identity for removal. Four overloads
differ only in where the effect's strength comes from:

1. **no strength** — the animation plays at full weight for its own length;
2. **a constant factor**, clamped to a range that allows slight *over*-application
   (up to one and a half) as well as attenuation;
3. **a strength function** — evaluated every frame, so the effect's intensity tracks a
   live game value. This is how drunkenness gets stronger as alcohol rises without anyone
   re-installing the effector;
4. **a controller object** — the effect's strength *and* its lifetime come from an object
   the caller owns, so the caller can end the effect at will.

**Notes** — the four overloads are four copies of the same twenty lines. A rebuild should
write one installer taking a strength source that is one of {none, constant, function,
controller}; nothing is lost.

## `RemoveEffector`

**Contract** — removes both halves by type. Removing a type that is not installed is a
no-op, so callers need not track what they added.

## `CAnimatorCamEffector`

**Contract** — plays a recorded camera animation. Each frame it reads the animation's
current transform and *composes* it onto the camera pose the stack has produced so far,
then advances the animation. It is valid — that is, stays on the stack — for the
animation's length, or forever when cyclic.

**Invariants** — the composition treats the animation as a **relative** motion by default:
the incoming camera pose is rebuilt as a basis and the animation's transform is applied
within it, so a shake shakes wherever the player is looking. An absolute-positioning flag
switches this to *replacement*, and that is what a scripted cutscene uses: the animation
then carries the camera's world position and orientation outright, and the player's own
aim is ignored.

**Notes** — the animation's transform is sampled *before* it is advanced, so the pose
applied on a given frame is the one from the previous frame's time. This is a one-frame
lag that nobody would notice and that a rebuild is free to fix.

The effector can also override the field of view, but only when the override is positive;
a negative value means "leave it alone". Using the sign of a value as a presence flag is
incidental — a rebuild should make it optional.

## `CAnimatorCamLerpEffector`

**Contract** — the same, blended. The animation's effect is computed exactly as above and
then interpolated between "no effect" and "full effect" by the strength function's current
value, clamped to the unit range.

**Invariants** — the orientation is interpolated as a **rotation**, not component-wise:
both the unmodified and the modified basis are converted to rotations and interpolated
spherically, and only the position is interpolated linearly. Interpolating the basis
vectors directly would shrink and skew the camera's frame at intermediate strengths.

```text
FUNCTION process(camera_info) -> bool
  base   = basis built from camera_info (up, direction, their cross, position)
  moved  = base composed with the animation's current transform
  advance the animation by the frame time

  t = strength(), clamped to 0..1
  orientation = spherical_interpolate(rotation_of(base), rotation_of(moved), t)
  position    = linear_interpolate(camera_info.position, moved.position, t)
  write orientation and position back into camera_info
  IF the field-of-view override is set THEN write it too
  RETURN true
```

**Notes** — the constant-strength variant is the same class with a strength function that
returns a stored number, wired to itself at construction. That self-binding is a C++
trick; a rebuild expresses it as a variant of the strength source.

## `CCameraEffectorControlled`

**Contract** — a blended effector whose strength *and* validity come from a controller
object. It registers itself with the controller on construction and clears that
registration on destruction, which is the other half of the controller's
outlive-me assertion. It stays on the stack exactly as long as the controller says it is
valid.

## `SndShockEffector`

**Contract** — the deafening effect of an explosion close by. It does three things at
once: it installs the paired camera-and-screen hit effector, it drops the global sound
volume to a tenth, and it ramps that volume back up over the length of a supplied sound
(the ringing-ears sound the caller plays). Its own strength decays linearly from the
moment it starts.

```text
Start(actor, sound_length, power):
  power clamped to 0.1 .. 1.5
  remember the current global sound volume, once      # only on the first start
  global sound volume = remembered * 0.1
  life_time = power * 4                               # 6 seconds at full power of 1.5
  end_time  = now + life_time
  install the "snd_shock_effector" section, controlled by this

Update():                                             # ramp the volume back
  elapsed = elapsed + frame_milliseconds
  x = elapsed / sound_length
  y = 2x - 1                                          # silent for the FIRST HALF, then ramp
  IF y > 0 THEN global volume = lerp(muted, remembered, y)

GetFactor():                                          # effect strength, 1 -> 0
  clamp((end_time - now) / 8, 0, 1)
```

**Invariants** — the stored volume is captured only the first time and restored in the
destructor unconditionally, so overlapping explosions do not ratchet the volume down
permanently. The destructor also removes the effector and then asserts that both effector
back-pointers are clear.

**Notes**

- The volume stays fully muted for the *first half* of the ringing sound and ramps over
  the second half. That is the effect: a moment of near-silence, then hearing returning.
- The strength divisor of eight and the six-seconds-at-full-power constant have no
  derivation; they are tuning. The divisor makes the visual effect finish well before the
  deafness does, at any power below the maximum.
- The effector *validity* and the volume ramp run on two different clocks — the ramp on
  accumulated frame milliseconds against the sound's length, the strength on the global
  clock against its own life time. They are not synchronized, and the effect is that the
  screen recovers and the hearing recovers independently.

## `CControllerPsyHitCamEffector`

**Contract** — the psychic attack: the camera is torn from the player and flown along a
straight line toward the attacker while the field of view narrows and the view jitters.
Its duration is fixed at construction; its lifetime on the stack is unbounded, because the
caller removes it.

```text
construct(from, to, duration, fov_from, fov_to):
  direction = normalize(to - from); distance = |to - from|
  jitter target = a small random angle on each of three axes

process(camera_info):
  ease each jitter angle toward its target at a fixed angular speed;
    when one arrives, pick a new random target on that axis
  progress = min(elapsed / duration, 1)
  position = from + direction * distance * progress
  field of view = interpolate(fov_from, fov_to, progress)
  orientation  = the direction toward the attacker, jittered
  once the duration is exceeded the jitter is dropped and the view is steady
```

**Invariants** — the camera's *direction* is the attack direction from the first frame, not
the player's; the player has lost control of the view entirely. The jitter is a random
walk within half a degree on each axis, which is small enough to read as an unsteady head
rather than as a shake.

**Notes** — the jitter continues right up to the end and then stops abruptly rather than
fading. The commented-out constants beside the class show an earlier version where the
field of view was hard-coded rather than supplied; the parameterized form is what ships.

## `CActorCameraManager`

**Contract** — the actor's effector stack, extended with the second *weapon camera* pose.
Before the stack runs, the weapon pose is seeded from the real one. Each effector that
succeeds is asked whether it affects the weapon camera; if it does, the **difference** it
made to the real camera is added to the weapon camera too. Field of view, far plane and
aspect are always copied across, because those are properties of the projection rather
than of the pose.

**Invariants** — after the stack has run, the weapon camera's basis is **re-orthonormalized**:
direction and up are normalized, the right vector is rebuilt from their cross product, and
up is rebuilt from the other two. It must be, because it was built by *adding differences*
of basis vectors, which does not preserve orthonormality. That re-orthonormalization is the
price of the difference-accumulation approach and a rebuild must either pay it or
accumulate rotations properly instead.

**Notes** — accumulating differences rather than re-running each effector against the
weapon pose means an effector is evaluated once, which matters because several of them are
stateful (they advance an animation). A rebuild that evaluates twice will double the
animation's speed.
