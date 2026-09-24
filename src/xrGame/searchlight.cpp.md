# src/xrGame/searchlight.cpp

> A steerable spot light on a two-axis mount: aims at a script-given target, moves both axes so they arrive together, and animates its colour.

**Needs** — [`searchlight.h`](searchlight.h.md) · [`script_object.h`](script_object.h.md) · [`script_entity_action.h`](script_entity_action.h.md) · [`script_watch_action.h`](script_watch_action.h.md) · [`xrEngine/LightAnimLibrary.h`](../xrEngine/LightAnimLibrary.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`game_object_space.h`](game_object_space.h.md) · [`xrServerEntities/xrServer_Objects_ALife.h`](../xrServerEntities/xrServer_Objects_ALife.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: owns renderer light and glow handles whose release is ordered against the object's teardown

## Purpose

The searchlight is the engine's only *aimable* light source. It is an entity with a skinned
visual: two bones form a yaw/pitch mount, a third bone is the lamp head, and a spot light
plus a glow billboard are pinned to that head every frame. Scripts drive it with the same
look order they give a creature, and it reports whether it has finished sweeping so a
script can wait on it.

Everything it needs beyond geometry — which bones, what colour, what cone, which textures
— is read from the *visual's own user data*, not from the entity's configuration section.
That is the notable structural decision: the mount is a property of the model, so a
level can place several differently-shaped searchlights of the same class.

## State

See [`searchlight.h`](searchlight.h.md). Two invariants hold across the file:

- `start` is captured once, at spawn, from the model's authored orientation, and never
  changes. Every aim is expressed as a signed offset from `start`, which is what lets the
  mount be clamped to a quarter turn either way without knowing anything about the level.
- `current` moves toward `target` and is the only pair the renderer sees. Nothing writes
  `current` directly except the per-frame interpolation.

## Spawn

**Contract** — refuses to spawn unless the server record really is a projector record and
unless the visual is a skinned model, because both the bone lookups and the mount depend
on it. Then, from the visual's user data section `projector_definition`:

```text
FUNCTION net_spawn(server_record) -> bool
  REQUIRE server_record IS projector record
  IF NOT base.net_spawn(server_record) THEN RETURN false
  REQUIRE visual IS skinned

  user_data = visual.user_data            # FAIL WITH "empty projector user data" if absent
  colour_anim = light_anim_library.find(user_data.color_animator)
  guide_bone  = visual.bone_id(user_data.guide_bone)
  bone_yaw.id = visual.bone_id(user_data.rotation_bone_x)
  bone_pitch.id = visual.bone_id(user_data.rotation_bone_y)

  colour      = user_data.color
  brightness  = colour.intensity          # cached: the animator supplies hue, this the level
  light.colour = colour
  light.range  = user_data.range
  light.cone   = radians(user_data.spot_angle)
  light.texture = user_data.spot_texture
  glow.texture = user_data.glow_texture
  glow.colour  = colour
  glow.radius  = user_data.glow_radius

  visible = true ; enabled = true
  turn_on()

  # the mount: two bones get a custom transform hook, applied after animation
  bind_bone_hook(bone_yaw.id,   yaw_hook,   this)
  bind_bone_hook(bone_pitch.id, pitch_hook, this)

  start = current = target = orientation_of(direction)
  RETURN true
```

**Invariants** — the bone hooks are bound *after* the base spawn, because the base is what
instantiates the visual; and `start` is captured *after* the hooks are bound but before any
update runs, so the first frame's hooks see a zero offset.

## The two bone hooks

**Contract** — the engine calls these while composing the skeleton's pose, after animation
and before skinning, with the bone instance to modify. Each post-multiplies an extra
rotation onto the animated transform, so the mount's sweep composes with whatever the
model's own animation does.

```text
FUNCTION pitch_hook(bone)                 # bound to the bone named rotation_bone_x
  bone.transform = bone.transform * rotation_from(heading=0, pitch=current.pitch, bank=0)

FUNCTION yaw_hook(bone)                   # bound to the bone named rotation_bone_y
  delta = angular_distance(start.yaw, current.yaw)
  IF signed_normalize(start.yaw - current.yaw) > 0 THEN delta = -delta
  bone.transform = bone.transform * rotation_from(heading=-delta, pitch=0, bank=0)
```

**Notes** — the naming is crossed and it is not a typo to fix blindly: the bone the data
calls `rotation_bone_x` receives the **pitch** and `rotation_bone_y` receives the **yaw**.
The names describe the axis each bone rotates *about* in the model's frame, not the angle
it is fed. A rebuild must keep the data key names, since they are in the shipped models'
user data.

The yaw hook is the only place `start` is used at run time: the pitch is absolute (the lamp
tilts the same way regardless of where the mount was authored pointing) but the yaw is
relative to the authored rest orientation, because the model's own animation already
contains that rest heading.

## On and off

**Contract** — switching on activates both light and glow and makes the lamp-head bone
visible, then forces a full bone recomputation immediately so the light is placed
correctly on the very frame it appears rather than one frame late. Switching off
deactivates both and hides the bone. Both are idempotent, guarded on the light's current
state — the guard matters because the on path costs a forced skeleton evaluation.

## Per-frame update

**Contract** — runs every frame, not on the scheduler, because the beam must not visibly
step.

```text
FUNCTION update_client()
  base.update_client()
  IF light.active
    IF colour_anim EXISTS
      c = colour_anim.sample(global_time)        # the library answers in blue-green-red order
      colour = (c.blue, c.green, c.red) scaled by brightness / 255
      light.colour = colour ; glow.colour = colour

    m = object_transform * bone_transform(guide_bone)
    light.orientation = (forward: m.forward, right: m.right)
    light.position = m.origin
    glow.position  = m.origin
    glow.direction = m.forward

  angle_lerp(current.yaw,   target.yaw,   bone_yaw.speed,   frame_delta)
  angle_lerp(current.pitch, target.pitch, bone_pitch.speed, frame_delta)
```

**Notes** — the colour animation library returns packed components in blue-green-red
order and this code reads them back in that order into a red-green-blue colour; the swap
is the un-packing, not a bug. The division by 255 converts the packed byte range to the
renderer's normalized range, and the multiplication by the cached authored intensity is
what lets one shared colour animation drive lamps of different brightness.

The angle interpolation runs whether or not the light is on, so a searchlight that is
switched off still tracks and is already aimed when it comes back on.

## Aiming: the watch action

**Contract** — when a script hands the searchlight a look order, this converts it into a
target orientation and a pair of axis speeds, and reports whether the sweep is still
running.

```text
FUNCTION assign_watch(entity_action) -> bool     # true while still turning
  IF NOT base.assign_watch(entity_action) THEN RETURN false
  order = entity_action.watch_order

  IF order.object_to_watch EXISTS
    set_target(order.object_to_watch.position)
  ELSE
    set_target(order.target_point)

  delta_yaw   = angular_distance(current.yaw,   target.yaw)
  delta_pitch = angular_distance(current.pitch, target.pitch)

  bone_yaw.speed = order.velocity_yaw
  time = delta_yaw / bone_yaw.speed             # time the yaw axis will take
  bone_pitch.speed = IF time is zero THEN order.velocity_pitch
                                     ELSE delta_pitch / time

  order.completed = (delta_yaw < epsilon) AND (delta_pitch < epsilon)
  RETURN NOT order.completed
```

**Invariants** — the load-bearing decision is the pitch speed. The yaw axis runs at the
speed the script asked for; the pitch axis is re-timed so that it finishes at the *same
moment*. Without this the beam would trace an L — sweeping across, then tilting — instead
of a straight sweep to the target. The script's pitch speed is used only when the yaw has
nothing to travel and the re-timing would divide by zero.

Completion is decided *before* the move, against the current orientation, so the order is
reported finished on the first evaluation after the beam has arrived.

## Aiming: the target setter

**Contract** — converts a world position into a clamped target orientation:

```text
FUNCTION set_target(position)
  (heading, pitch) = orientation_of(position - self.position)

  delta = angular_distance(heading, start.yaw)
  IF signed_normalize(heading - start.yaw) > 0 THEN delta = -delta
  clamp delta to [-quarter turn, +quarter turn]
  target.yaw = normalize(start.yaw + delta)

  clamp pitch to [-quarter turn, +quarter turn]
  target.pitch = pitch
```

**Invariants** — yaw is clamped *relative to the rest orientation*, so a searchlight can
never swing more than a quarter turn from where the level author pointed it. That is what
keeps the mount from turning through its own pillar, and it is why `start` has to be kept.
Pitch is clamped absolutely, straight up to straight down.

**Notes** — the sign dance around the angular distance is the same in both the target
setter and the yaw bone hook: the distance function answers an unsigned magnitude and the
signed normalization recovers which way round. A rebuild with a signed angular-difference
primitive collapses both to one subtraction.

## Other lifecycle answers

**Contract** — loading from a configuration section and the scheduled update both delegate
to the scripted-object base unchanged; they are overridden only because the base declares
them. The object action handler honours exactly two goals — turn on, turn off — and
delegates the rest. Navigation locations are declined outright: a searchlight has no place
on the navigation mesh and creatures must not path to it.
