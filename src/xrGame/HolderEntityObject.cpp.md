# src/xrGame/HolderEntityObject.cpp

> A generic mountable object: a static prop the player can occupy, which takes over the camera and the input while occupied. The minimum viable holder, with the weapon and driving behaviour deliberately absent.

**Needs** — [`HolderEntityObject.h`](HolderEntityObject.h.md) · [`holder_custom.h`](holder_custom.h.md) · [`PhysicsShellHolder.h`](PhysicsShellHolder.h.md) · [`Actor.h`](Actor.h.md) · [`CameraFirstEye.h`](CameraFirstEye.h.md) · [`ActorEffector.h`](ActorEffector.h.md) · [`Level.h`](Level.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrPhysics/PhysicsShell.h`](../xrPhysics/PhysicsShell.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — reached through its declarations in [`HolderEntityObject.h`](HolderEntityObject.h.md); callers name that, not this file.
**Tier floor** — T2: camera transform composition and input forwarding

## Purpose

The engine has a general notion of a **holder** — something the player can get into, which
then owns the camera, receives the input, and decides whether a weapon may be carried. The
vehicle is the elaborate implementation of it. This is the plain one: a prop bolted to the
world that the player can sit at.

Its value to a rebuild is that it shows the holder contract *stripped of everything else*.
What a holder must do is exactly what this file does and no more:

- place a camera at a configured offset from its own transform, and drive it from the
  mouse;
- turn the occupant's head to match, so the body's animation agrees with the view;
- declare whether a weapon may be held and where the player is put down on exit;
- refuse damage while occupied, because the occupant is the thing that should be hit.

Several methods are present, empty. That is not incompleteness to be tidied away — they
are the holder interface's full surface, and their emptiness *is* this holder's answer.

## State

```text
RECORD HolderEntityObject
  camera          : Camera          # first-person, rigidly linked to this object's transform
  camera_position : vec3            # the eye point, in the object's own space
  camera_angle    : vec3            # the rest orientation of the eye, in the same space
  exit_position   : vec3            # where the occupant is placed on dismount
  allow_weapon    : bool            # may the occupant hold a weapon while mounted
  exit_locked, enter_locked : bool  # script gates on mounting and dismounting
  use_hint        : optional<text>  # the crosshair caption offering the action
```

Invariants:

- The camera is constructed *with* the object and lives as long as it, not as long as the
  occupancy. Mounting binds the existing camera into the actor's camera stack; it does not
  create one.
- The camera is declared rigidly linked in both position and direction, meaning it follows
  the object's transform exactly with no spring or lag. A holder that moves would need a
  different choice; this one does not move.
- Damage is only taken while unoccupied.

## `Load`

**Contract** — reads whether a weapon is permitted, whether entering and exiting are
locked, the exit position, the camera's position and rest angle, and the use hint. Only
the first three are required; the geometry defaults to the object's origin, which places
the camera inside the prop — so a section that omits them is usable but wrong.

## `net_Spawn`

**Contract** — spawn, then build a physics body from the model with its **root bone
fixed**, and enable per-frame processing, visibility and collision.

**Invariants** — fixing the root is what makes this a prop rather than a loose object: it
has collision geometry and mass properties, and it never moves. A holder that should move
must not fix it.

Per-frame processing is enabled unconditionally at spawn and never conditioned on
occupancy, even though the only per-frame work is the camera update. That is a small
waste a rebuild can avoid by enabling it on mount.

## `cam_Update`

**Contract** — place the camera and turn the occupant's head. Transforms the configured
eye point through the object's transform into the world, forces the occupant's orientation
to the negated camera angles, sets the field of view, updates the camera at the computed
position and rest angle, and pushes the result into the level's camera.

**Invariants** — the occupant's yaw and pitch are the camera's *negated*. The two
coordinate conventions disagree in sign and this is where the disagreement is reconciled;
getting it wrong makes the body's head turn the wrong way while the view is correct, which
is only visible to other players.

```text
FUNCTION camera_update(dt, field_of_view)
  eye = object transform applied to the configured camera position
  IF occupied THEN
    occupant.orientation.yaw   = -camera.yaw
    occupant.orientation.pitch = -camera.pitch
  camera.field_of_view = field_of_view
  camera.update(eye, configured rest angle)
  push the camera into the level's active camera
```

## `UpdateCL`

**Contract** — each frame, if this holder is occupied *and its camera is the one the
occupant is currently looking through*, run the camera update and apply the result to the
device.

**Invariants** — the second condition matters: an occupant may be looking through
something else (a scope, a scripted camera), and the holder must not fight for the view.

## `OnMouseMove`

**Contract** — turn the camera by the mouse delta, scaled by the user's sensitivity
setting and by the **ratio of the current field of view to the default**. Ignored on a
client mirror. The vertical axis is additionally scaled by three quarters and honours the
invert setting.

**Invariants** — scaling by the field-of-view ratio is what keeps the mouse feeling the
same when zoomed: a narrower view turns proportionally slower, so the same hand movement
covers the same distance on screen.

**Notes** — the three-quarters factor on the vertical axis is a fixed aspect compensation
and the divisor of fifty on the sensitivity is an unexplained scale. Both are inherited
from the actor's own mouse handling and must match it, or switching between the actor and
a holder changes the mouse feel.

## `attach_Actor` / `detach_Actor` / `attach_actor_script` / `detach_actor_script`

**Contract** — mounting delegates to the generic holder and then disables the physics
body's callbacks; dismounting reverses both. The script forms go through the actor, which
is the object that actually tracks what it is occupying, and take a force flag that
bypasses the enter and exit locks.

**Invariants** — the callback toggle is the whole meaning of `SetBoneCallbacks` here:
while occupied, the body stops reporting contacts, so the prop does not react physically
to the occupant standing on it. The names are inherited from the vehicle, where they do
install per-bone callbacks; here they do not, and a rebuild should rename them to what
they are.

## `Hit`

**Contract** — damage is passed to the base only while unoccupied. An occupied holder
absorbs nothing; the occupant is the target.

## `Use`

**Contract** — this holder may be used exactly when it is unoccupied. It ignores the
position, direction and foot position the interface supplies, which a vehicle uses to
decide *which door* the player is reaching for.

## `allowWeapon`, `HUDView`, `ExitPosition`, `Camera`, `GetInventory`

**Contract** — the configured weapon permission; always first-person; the configured exit
point; the camera; and no inventory of its own — an occupant keeps their own.

## `Action`, `SetParam`, `OnKeyboardPress`, `OnKeyboardRelease`, `OnKeyboardHold`, and the four controller hooks

**Contract** — forwarded to the generic holder or empty. This holder has no controls. The
fire command is matched and ignored in both the press and release handlers, and a
commented-out block in the action handler shows where a mounted weapon's fire and cease-fire
would go.

**Notes** — these empty handlers *are* the contract for a rebuild: a holder must accept
every input event and is free to discard it. The keyboard handlers return early on a
client mirror, which the empty controller handlers do not — an inconsistency with no
effect, since neither does anything.

## `renderable_Render`, `net_Export`, `net_Import`, `net_Destroy`

**Contract** — pure delegation, plus disabling per-frame processing on teardown.

## `BoneCallbackX` / `BoneCallbackY`

**Contract** — declared and empty. They exist because the vehicle this class was derived
from installs turret-aiming callbacks on two bones; this holder has no turret. Nothing
references them.
