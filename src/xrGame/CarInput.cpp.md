# src/xrGame/CarInput.cpp

> The vehicle's control surface: which action does what at the wheel, how an analogue stick becomes the same three-valued steering a key produces, and how a script drives a car with no driver.

**Needs** — [`Car.h`](Car.h.md) · [`Actor.h`](Actor.h.md) · [`CarWeapon.h`](CarWeapon.h.md) · [`car_memory.h`](car_memory.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`script_entity_action.h`](script_entity_action.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`Level.h`](Level.h.md) · [`CameraLook.h`](CameraLook.h.md) · [`CameraFirstEye.h`](CameraFirstEye.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: action dispatch and a little sensitivity arithmetic

## Purpose

Three things that all reduce to "something outside the vehicle wants to drive it".

A **player** presses actions. A **script** hands the vehicle an action record describing
which movement keys it wants held. A **controller** supplies two continuous sticks that
have to be reduced to the same commands. All three funnel into the same press/release
entry points, which is what guarantees that a scripted car and a driven car behave
identically — a real requirement, because the games use scripted vehicles in sequences the
player watches.

The action vocabulary is the engine's, not this file's. Nothing here reads a key; it reads
`kFWD`, `kACCEL`, `kTORCH` and so on. That is why the *same* actions mean walking when the
player is on foot and driving when they are in a car: the binding layer knows nothing about
either.

## State

`Stateless.`

## The action table

**Contract** — the mapping from action to vehicle operation, on press and on release. This
table is the file's real content.

| Action | Press | Release |
|---|---|---|
| first / second / third camera | select that camera | — |
| accelerate | shift up | — |
| crouch | shift down | — |
| forward | accelerator | release accelerator |
| back | reverse (brakes first) | release reverse |
| strafe right / left | steer right / left, and tell the rider to lean | centre the steering, straighten the rider |
| jump | handbrake | release handbrake |
| engine, or detector | toggle the engine | — |
| torch | toggle the headlights | — |
| use | nothing (the holder handles it) | — |
| zoom in/out, look up/down/left/right | move the active camera | — |

**Notes** — the reuse is the decision. Gear shifting rides on sprint and crouch; the
handbrake rides on jump; the headlights ride on the torch. No vehicle-specific bindings are
added at all, which means a player who has rebound their movement keys automatically has
rebound their driving. Two actions toggle the engine — the dedicated one and the detector
key — so that a build whose bindings predate the dedicated action still starts a car.

The steering actions also call the rider's own steering pose, so the driver is animated
turning the wheel.

Every entry point refuses to act on a vehicle this host does not own, so a remotely
simulated car ignores local input entirely.

## `OnAxisMove`, `OnMouseMove`, `OnControllerAttitudeChange`

**Contract** — convert a two-axis delta into camera movement, scaled by the player's
sensitivity settings and by how far the camera is zoomed in.

```text
FUNCTION axis_move(x, y, scale_x, scale_y, invert_x, invert_y)
  IF x is non-zero THEN move the camera left or right by |x * scale_x|, sign chosen by x
  IF y is non-zero THEN move the camera up or down   by |y * scale_y * 3/4|
```

**Invariants** — the scale always includes the ratio of the camera's current field of view
to the default. Zooming in must slow the pointer by the same factor it magnifies the
image, or aiming becomes impossible at high zoom. The vertical axis additionally carries a
fixed three-quarters factor, which is an aspect-ratio compensation baked in rather than
derived from the actual aspect.

## The controller path

**Contract** — one stick looks, the other drives. The driving stick is reduced to the same
discrete commands a keyboard produces, with a dead zone.

```text
FUNCTION on_move_stick(state)
  steer = state.x IF |state.x| > 0.35 ELSE 0      # dead zone
  Steer(steer)                                     # analogue, unlike the keyboard path
  tell the rider to lean by the same amount
  IF |state.y| > 0.35 THEN
    release whichever of forward/back is held in the wrong direction
    press the one the stick indicates
  ELSE release both
```

**Notes** — steering is genuinely analogue on a stick and genuinely three-valued on a
keyboard, and both paths end in the same `Steer`. The throttle is *not* analogue: past the
dead zone the stick is a switch. That asymmetry is deliberate — the accelerator's state
machine (see [`Car.cpp`](Car.cpp.md)) is built on held flags and has no notion of partial
throttle.

The dead zone is 0.35 on both axes, which is large; it has to be, because the same stick
must not steer while the player is pushing it forward.

An action on the controller that is neither stick falls through to the keyboard press
handler. The release handler does too — **it calls press, not release**, which means a
controller button that maps to an action with a meaningful release (forward, back, the
handbrake) latches on. That is a bug, and it is the shipped behaviour.

## `bfAssignMovement`

**Contract** — a script movement action names a set of input keys it wants held this
instant. Translate them into presses and releases. Returns whether the action is still
running.

```text
FUNCTION assign_movement(action) -> bool
  IF the action is already complete THEN RETURN false
  FOR EACH of forward, back, left, right, shift-up, shift-down, brakes
    press it if the action's key set contains it, release it otherwise
  IF the set asks for engine-on  THEN start the engine
  IF the set asks for engine-off THEN stop the engine
  RETURN true
```

**Notes** — the script layer speaks the same seven keys as the player, *including*
shift-up and shift-down, so a scripted car shifts gears. Engine on and off are separate
because there is no key for them in the script vocabulary and a toggle would be
non-idempotent, which a script replayed each frame cannot tolerate.

A commented-out line would have driven the engine speed directly from the action's
requested speed. It is disabled: a scripted car accelerates the same way a driven one
does, which is slower but does not fight the physics.

## `bfAssignObject`

**Contract** — a script object action names a *bone* and a verb. If the bone is a door,
open, close or toggle it; if the bone is a light, turn it on, off or toggle it; otherwise
do nothing. Returns whether the action is still running.

**Notes** — the action completes immediately for a light and continues for a door, because
a light switches instantly and a door takes time to swing. Naming the target by bone rather
than by an index is what lets level scripts address "the driver's door of this specific
car" without the script knowing the vehicle's internal ordering.

## `isObjectVisible`

**Contract** — can the vehicle see a given object? Two entirely different answers depending
on whether the model declared a visual memory.

```text
FUNCTION is_object_visible(target) -> bool
  IF this vehicle has a visual memory THEN
    RETURN whether that memory currently lists the target as visible
  ELSE
    from = this vehicle's centre, raised to the mounted weapon's height if it has one
    RETURN no static geometry blocks the segment from `from` to the target's centre
```

**Notes** — the fallback is a bare line of sight against *static* geometry only: other
vehicles, creatures and dynamic objects do not block it. It is good enough for a turret
deciding whether to bother traversing, and it is not a perception system. The memory-backed
path is the real one, and it exists so that a scripted gun car behaves like a creature —
with build-up, decay and a frustum — rather than snapping onto anything in line of sight.

## Delegations

**Contract** — `Action`, both `SetParam` overloads, `WpnCanHit`, `FireDirDiff` and
`HasWeapon` forward to the mounted weapon if one exists and answer harmlessly if not;
`CurrentVel` reports the assembly's linear velocity; `vfProcessInputKey` is the one-line
press-or-release switch the script path uses.
