# src/xrGame/CameraLook.cpp

> The three third-person cameras: one that orbits the player at a distance the world can push in, one that adds a shoulder offset and an auto-aim lock, and one that holds a fixed framing.

**Needs** — [`CameraLook.h`](CameraLook.h.md) · [`Actor.h`](Actor.h.md) · [`actor_memory.h`](actor_memory.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`xrEngine/CameraManager.h`](../xrEngine/CameraManager.h.md) · [`xrEngine/xr_level_controller.h`](../xrEngine/xr_level_controller.h.md) · [`xrEngine/xr_input.h`](../xrEngine/xr_input.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — reached through its declarations in [`CameraLook.h`](CameraLook.h.md); callers name that, not this file.
**Tier floor** — T2: a ray cast and quaternion interpolation every frame

## Purpose

Third-person viewing in this engine is one idea with three variations. The idea: the camera
looks at a point, sits some distance *behind* that point along its own view direction, and
that distance is shortened by whatever the world puts in the way — so the camera never ends
up inside a wall and never shows the player the inside of the world.

The three variations are the plain orbit, an offset orbit with a target lock, and a camera
that eases to a fixed framing and stays there.

## State

```text
RECORD LookCamera
  zoom_limits : (real, real)   # the authored minimum and maximum orbit distance
  distance    : real           # the requested distance; invariant: within the limits
  previous_d  : real           # the smoothed ACTUAL distance after collision
```

**Invariant** — the requested distance and the actual distance are separate. The player's
zoom moves the request; the world moves the actual. Conflating them would make the camera
permanently zoomed in after brushing a wall.

## `Load`

**Contract** — reads the two orbit limits and starts the camera at their midpoint. The
smoothed distance starts at zero, so the first frame pulls out from the player rather than
snapping.

## `Update` · `UpdateDistance`

**Contract** — builds the view basis from the three angles, then resolves the distance
against the world.

```text
FUNCTION update(point)
  basis = rotation from (-yaw, -pitch, -roll)
  direction, up = basis forward and up
  IF linked to the parent's frame THEN rotate both by the parent's transform
  resolve_distance(point)

FUNCTION resolve_distance(point)
  # Cast BACKWARD from the look-at point along the view direction.
  # The margin is six near-plane widths: the camera must stop short of the
  # wall by enough that its near plane is also clear.
  margin = near_plane * 6
  hit = raycast(from: point, along: -direction, max: distance + margin,
                against: static and dynamic, ignoring: the parent)

  # Smooth the result, so brushing past a doorway does not snap the camera.
  actual = inertia * previous_actual + (1 - inertia) * (hit.range - margin)
  previous_actual = actual
  position = point - direction * (actual + near_plane)
```

**Invariants** — the smoothing is exponential with a configurable weight and applies in
*both* directions: the camera pulls in quickly and eases back out at the same rate. That
symmetry is why the third-person view feels sluggish when leaving cover, and it is a
deliberate trade against snapping.

The ray ignores the parent — the player's own body must not push the camera away.

**Notes** — the margin's factor of six is unexplained and is the number that decides how far
from a wall the camera stops. It has to exceed the near plane's diagonal (see
[`ActorCameras.cpp`](ActorCameras.cpp.md), which computes exactly that quantity for the
lean test); six is a generous constant chosen instead of computing it.

## `Move`

**Contract** — zoom in and out move the requested distance; the four rotation directions
move yaw and pitch, at a rate divided by the caller's inertia factor. Angles are clamped to
their limits if installed and the distance is always clamped to the orbit limits.

**Notes** — the pitch and yaw rates are read from the *first two* components of the rate
vector here, while the first-person camera reads them in the opposite order. One of the two
is transposed relative to the shared configuration; the shipped data is tuned around it, so
reproduce it rather than harmonize it.

## `OnActivate`

**Contract** — inherits the outgoing camera's yaw *and position* when the two share a
reference space, then normalizes the yaw into one turn. Inheriting the position matters:
without it the orbit camera appears at the player's eye for one frame before the distance
resolves.

## `CCameraLook2` — the offset orbit with auto-aim

**Contract** — the same orbit, with two additions.

**A fixed shoulder offset.** The camera sits at an authored offset from the look-at point,
rotated by the camera's own yaw, rather than directly behind it. Note that this variant
**does not resolve the distance against the world at all** — the offset is applied outright.
A rebuild should check whether that is intended; it means this camera can be pushed into
geometry.

**A target lock.** While the auto-aim action is held, the camera picks the *nearest living
creature the actor can currently see*, taken from the actor's own visual memory, and eases
its aim onto that creature's centre plus a small upward offset. Releasing the action drops
the lock.

```text
FUNCTION update_autoaim()
  target = locked enemy's centre, raised by 0.2 m     # aim at the chest, not the navel
  desired = the yaw and pitch pointing from the camera to that point
  ease yaw   toward desired, with the authored minimum and maximum speeds
  ease pitch toward desired, likewise
```

**Invariants** — the candidate set is the actor's *visual memory*, filtered to those visible
*right now*. The lock therefore cannot acquire a creature the actor has not seen, and drops
nothing when the creature is merely remembered. The nearest is measured horizontally only,
so a creature directly above or below does not win by being close.

**Notes** — the auto-aim key is read as a **raw key state**, by enumerating every physical
key bound to the action and polling each. That is the correct way to ask "is this action
currently held" in a system that only delivers action events, and it is worth noting as the
pattern for any held-action query.

The shoulder offset is a **static** shared by every instance of this camera, loaded from
whichever section was loaded last. With one player that is harmless and in a rebuild it
should simply be a field.

## `CCameraFixedLook`

**Contract** — a camera that ignores input entirely and eases from its current orientation
to a fixed one over about a second, then holds it. Used where the game wants to frame a
scene.

```text
OnActivate: current = the incoming orientation
            final   = that orientation rotated a quarter turn about its own X axis
                      # the fixed framing is always a quarter-turn pitch from wherever
                      # the previous camera was looking
Update:     current = spherical_interpolate(current, final, frame_time)
            build the basis from it, place at the point, resolve the distance
Move:       does nothing
Set:        forces both current and final to the given angles, ending the ease
```

**Invariants** — interpolating by the *frame time* rather than by an accumulated fraction
makes this an exponential approach whose rate is one per second, which is why the comment
calls it one second. It never quite arrives, and nothing requires it to.

**Notes** — the activation converts between an angle triple and a rotation three times,
including a round trip through a matrix that exists only to re-extract the angles with
different signs. That is a sign-convention reconciliation written as arithmetic; a rebuild
with one convention deletes all of it.
