# src/xrGame/car_memory.cpp

> Gives a vehicle a pair of eyes, so that it can see the actor and nothing else.

**Needs** — [`car_memory.h`](car_memory.h.md) · [`vision_client.h`](vision_client.h.md) · [`Car.h`](Car.h.md) · [`Actor.h`](Actor.h.md)
**Used by** — reached through its declarations in [`car_memory.h`](car_memory.h.md); callers name that, not this file.
**Tier floor** — T2: a frustum description handed to the vision system

## Purpose

The vision half of the `feel` system needs a camera from whatever object is doing the
seeing: a position, an orientation and a frustum. A creature gets those from its head bone.
A vehicle has no head, so this supplies them explicitly — whoever drives the vehicle's
sensing (its turret, its script) sets the camera, and this object hands it to the vision
system.

It also narrows the vision to a single question. A vehicle does not need to know about
every creature in its field of view; it needs to know about the player. So it declares only
the actor relevant, and the vision system's per-frame raycasting cost collapses to at most
one candidate.

## State

```text
RECORD car_memory
  object          : reference to the vehicle
  view_position   : vector      # world; set by the vehicle, not derived from it
  view_direction  : vector      # defaults to forward
  view_normal     : vector      # defaults to up
  fov_degrees     : real        # from configuration
  aspect          : real        # from configuration
  far_plane       : real        # from configuration
```

**Invariants** — the camera is *pushed in*, never read from the vehicle's transform. A
vehicle whose camera is never set sees along the world's forward axis from the world origin,
which is the defaults' honest meaning: no camera, no vision. Nothing detects that case.

**Notes** — the vision client is constructed with a period of 100 milliseconds, so the
vehicle re-evaluates what it can see ten times a second rather than every frame. That is
the entire cost control: at one candidate per evaluation it is free.

## `reload`

**Contract** — reads the frustum from the vehicle's configuration section: field of view in
degrees, aspect ratio, and far plane. Three authored numbers, one per vehicle type, because
a vehicle's useful sight range is a property of what it is for.

## `camera`

**Contract** — hands the vision system the camera. Position, direction and normal are the
pushed-in values verbatim; the field of view is converted from the authored degrees to
radians; the near plane is fixed at a tenth of a metre.

**Notes** — the near plane is the one value not authored. Nothing can be within ten
centimetres of a vehicle's sensor and matter, so it is a floor rather than a tuned number.

## `set_camera`

**Contract** — records the camera for the next evaluation. The only writer.

## `feel_vision_isRelevant`

**Contract** — true only for the actor. This is the vision system's admission filter,
applied before any frustum or raycast work, and returning false for everything else is what
keeps a vehicle's vision from costing anything in a level full of creatures.

**Notes** — the file's original form returned false unconditionally, and that line is still
present, disabled, beside the current one. The history matters: vehicle vision was switched
off entirely at some point and then re-enabled for the actor alone. A rebuild should read
this as "vehicles see only the player, deliberately", not as an unfinished generalization.
