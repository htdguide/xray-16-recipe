# src/xrGame/WeaponHUD.h

> The superseded first-person weapon model: a shared animated visual, four measured points on it, and an animation timer that calls back when a motion ends.

**Needs** — [`Include/xrRender/KinematicsAnimated.h`](../Include/xrRender/KinematicsAnimated.h.md) · [`hud_item_object.h`](hud_item_object.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a shared-resource cache and an animation clock; nothing here touches a byte layout.

## Purpose

This is the older of two first-person weapon rigs. The one in use is
[`player_hud.h`](player_hud.h.md), which supports two items in hand and per-item measured
offsets; this one supports one item and is retained because parts of the weapon
hierarchy still name its types. It is a header with no implementation file in this
directory — the substance is the contract, which is why it gets the full treatment.

Three ideas are worth carrying forward even though the class is not:

1. **The first-person model is shared between every instance of a weapon type.** A
   rifle's view model is loaded once, keyed by section name, and reference-counted; each
   weapon holds a handle. That is why a hundred rifles cost one skeleton.
2. **The muzzle, the second muzzle and the ejection port are measured off a bone, not
   authored as world offsets.** The section names a *fire bone*; the three points are
   offsets in that bone's frame. A weapon whose animation moves the barrel therefore
   fires from where the barrel actually is.
3. **Animation end is an event, not a poll.** The rig holds the end time of the running
   motion and fires a callback into the weapon when it passes — which is the coupling the
   whole firing state machine's transitions depend on (see
   [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md)).

## State

```text
RECORD SharedWeaponVisual            # one per weapon section, reference counted
  animations   : AnimatedVisual      # the skinned first-person model
  fire_bone    : int                 # the bone the three points are measured against
  fire_point, fire_point_2, shell_point : vector3   # offsets in that bone's frame
  offset       : matrix              # model -> eye

RECORD WeaponHud
  owner        : HudItem
  shared       : handle to SharedWeaponVisual
  transform    : matrix              # where the model is this frame
  hidden       : bool                # explicitly stowed
  visible      : bool                # drawn at all
  anim_end_time : int                # when the running motion finishes
  stop_at_end_running : bool         # whether that end fires a callback
  started_anim_state : int           # the weapon state the motion was started for
  callback_item : HudItem            # who to tell
  zoom_rotate_x, zoom_rotate_y : real
  zoom_offset  : vector3
```

**Invariants** — the callback carries the state the animation was *started* for, not the
state the weapon is in when it ends. That is what lets a transition arriving mid-animation
be distinguished from the animation it was waiting on.

## The shared container

**Contract** — a process-wide, reference-counted table keyed by section name. Requesting
a section either returns the existing entry or loads one; the last release frees it.
Three static entry points bring it up, tear it down, and drop entries with no live
references.

```text
FUNCTION create(handle, section, owner)
  IF the container has no entry for section THEN
    build one by loading the model named by that section and measuring its
    fire bone and three points
  handle = a counted reference to that entry
```

**Notes** — the owner is passed to the loader so it can report which weapon caused a
failure. A rebuild's cache needs no such parameter.

## Animation

**Contract** — three entry points:

- `animPlay(motion, mix_in, callback_item, state)` — start a motion, optionally blending
  into the current pose, and record that when it ends `callback_item` should be told,
  tagged with `state`. A zero callback item means "play it and forget".
- `animDisplay(motion, mix_in)` — start a motion with no callback.
- `animGet(name)` — resolve a motion by name within this weapon's bank.

`Update` advances the clock and, when the end time passes, stops the motion and fires the
callback. `StopCurrentAnimWithoutCallback` stops it and suppresses the callback, which is
how the hidden-state transition avoids re-entering the state machine.

`random_anim(motions)` picks one of up to eight variants uniformly — how a weapon gets
several idle or reload animations without the caller choosing.

## Measured points and transform

**Contract** — `FireBone`, `FirePoint`, `FirePoint2` and `ShellPoint` expose the shared
entry's measurements; `Transform` is the per-frame placement, written by
`UpdatePosition`. `Visual` exposes the renderable so the renderer can draw it.

The zoom rotation and offset are the aim-pose adjustment, superseded by the two-pose
interpolation in [`Weapon.cpp`](Weapon.cpp.md).

## Visibility

**Contract** — two independent flags. `hidden` means the weapon stowed itself (during a
holster, or while a full-screen scope picture replaces the model); `visible` means the
renderer should consider it at all. Both must be true for the model to appear.

**Notes** — the debug-only setters for the three measured points write *through* the
shared entry, so moving the muzzle in the tuning tool moves it for every instance of that
weapon at once. That is intended — it is how the points were authored in the first place.
