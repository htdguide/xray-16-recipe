# src/xrGame/stalker_animation_callbacks.cpp

> Aiming: three bones — head, shoulder, spine — are rotated toward the sight target after the pose has been computed, with the weapon's recoil layered on, and with a slerp fade when a whole-body animation is taking the body over.

**Needs** — [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`EffectorShot.h`](EffectorShot.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: runs inside the renderer's pose evaluation, per bone per frame, on the skinning path

## Purpose

A stalker aims by *bending*, not by playing an aim animation: the authored motions face
straight ahead, and the head, shoulder and spine are twisted afterwards to point the weapon
where the sight system decided. That "afterwards" is the whole difficulty — the pose is
computed inside the renderer, after the game layer has finished its frame, so the correction
has to be installed as a callback the renderer invokes per bone while it walks the skeleton.

This file is those callbacks, the parameter records that carry the game layer's answers into
them, and the two modes they run in.

## State

```text
RECORD callback_params           # one per aimed bone, owned by the animation manager
  rotation : reference to a rotation the sight system keeps current
  object   : reference to the stalker
  blend    : optional reference to the whole-body blend to fade against
  forward  : bool                # fade in (true) or out (false)
```

**Invariants** — every field is a *reference into live state*, not a snapshot. The rotation
is the sight system's own, updated every frame; reading it inside the renderer is how the
correction stays current without the game layer pushing it. The lifetime rule is therefore
strict: these records are fields of the animation manager, the manager outlives the
skeleton's callbacks, and the callbacks are cleared before the manager goes away.

The bones are named in the stalker's configuration section — `bone_head`, `bone_shoulder`,
`bone_spin` — and resolved to indices at installation. A model whose rig names its bones
differently supplies its own section; the names are data, not code.

## `callback_rotation` — the direct mode

**Contract** — rotate one bone toward the sight target and layer the weapon's recoil on top,
preserving the bone's translation. Runs inside the renderer's pose walk. Does nothing when
the sight system is disabled.

```text
FUNCTION callback_rotation(bone)
  IF the stalker's sight system is disabled  RETURN     # dead, or not aiming at anything

  position = bone.transform.translation          # remember it: we only want the rotation
  bone.transform = rotation THEN bone.transform  # apply the aim in the bone's parent frame

  recoil = the weapon shot effector
  IF recoil is not active
    bone.transform.translation = position        # restore, and we are done
    RETURN

  angles = recoil.current delta angles, each normalized to a signed range
  # In cover the recoil is suppressed entirely; otherwise the BONES take a tenth of it.
  angles = angles * 0            IF the stalker is in cover
           angles * 0.1          OTHERWISE
  bone.transform = rotation_from(angles) THEN bone.transform
  bone.transform.translation = position
```

**Invariants** — the translation is saved and restored around both rotations. Composing a
rotation in the parent frame moves the bone's origin as well as its orientation, and a moved
head bone detaches the head from the neck. The correction is *orientation only*, always.

The angles are normalized to a signed range before scaling, or a recoil that has wrapped
past a full turn would be scaled to a tenth of a large number instead of a tenth of a small
one.

**Notes** — the recoil fraction is the interesting constant. The full recoil goes to the
weapon and the camera; the *bones* take a tenth of it, which is what makes a firing stalker's
upper body shudder visibly without its head snapping around. In cover the fraction is zero,
because a stalker leaning out of cover is posed by the cover's own authored animation and
adding recoil to that would push the body through the geometry it is hiding behind.

## `callback_rotation_blend` — the fading mode

**Contract** — the same correction, faded in or out along the progress of the whole-body
blend, with no recoil. Runs in the same place.

```text
FUNCTION callback_rotation_blend(bone)
  fraction = blend.time_elapsed / blend.time_total   IF a blend exists ELSE 1
  fraction = fraction IF fading forward ELSE (1 - fraction)

  # Interpolate along the shortest arc from no rotation to the full aim.
  rotation = slerp(identity, aim_rotation, fraction)

  position = bone.transform.translation
  bone.transform = rotation THEN bone.transform
  bone.transform.translation = position
```

**Invariants** — the fade is a **spherical interpolation between the identity and the full
rotation**, not a scaling of the rotation's Euler angles. The original contains both and the
Euler version is disabled, which is the right call: scaling three angles interpolates along a
path that is not the shortest arc, and for a large aim offset the head visibly swings out and
back on the way. A rebuild must use the arc.

The fraction is clamped by construction to the unit range and is checked.

**Notes** — this mode exists for the hand-off to a whole-body animation. While a global
motion is taking over, the aim correction has to come off — the motion poses the whole body
itself and a superimposed twist fights it — but coming off instantly snaps the head. So the
correction fades out over exactly the blend that is bringing the motion in, and fades back in
over the blend that takes it away. The direction flag is which of the two is happening.

The recoil is dropped in this mode entirely, because a stalker transitioning into a scripted
whole-body motion is not shooting.

## `assign_bone_callbacks`, `assign_bone_blend_callbacks`

**Contract** — install the direct or the fading callback on all three bones, pointing each
bone's parameter record at the sight system's corresponding rotation. The fading form also
points every record at the global channel's blend and records the fade direction. Reads the
bone names from the stalker's configuration section. Hard-fails if the visual is not a
skeleton.

**Invariants** — the three bones take three *different* rotations — head, shoulder and spine
each have their own — and the sight system distributes one aim direction across them. That
distribution, not this file, decides how much of a turn is neck and how much is waist.

Installing one mode overwrites the other on the same bones, which is how the transition
works: there is no state machine, only whichever callback was installed last.

## `remove_bone_callbacks`

**Contract** — clear the callback and the parameter pointer on all three bones. Must run
before the parameter records can go away.

## `clear_unsafe_callbacks`

**Contract** — if the bones are currently in fading mode, put them back into direct mode. A
no-op otherwise.

**Invariants** — this is the safety catch for the ladder in
[`stalker_animation_manager_update.cpp`](stalker_animation_manager_update.cpp.md). The fading
callbacks hold a reference to the global channel's blend; if the global channel is reset
while they are installed, that reference dangles. Every path that resets the global channel
calls this first. A rebuild that keeps the fade must keep this pairing, or find a way for the
callback to tolerate a blend that has gone away.

## `forward_blend_callbacks`, `backward_blend_callbacks`

**Contract** — report which mode and direction the bones are in, by inspecting the head
bone's record. Both answer false in direct mode.

**Notes** — the head bone stands for all three, which is correct only because all three are
always installed together. That coupling is not enforced anywhere.
