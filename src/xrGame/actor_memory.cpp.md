# src/xrGame/actor_memory.cpp

> The player's vision: what the player has actually seen, computed with the player's own camera, so that scripts and creatures can ask what the human is looking at.

**Needs** — [`actor_memory.h`](actor_memory.h.md) · [`Actor.h`](Actor.h.md) · [`GamePersistent.h`](GamePersistent.h.md) · [`vision_client.h`](vision_client.h.md) · [`xrEngine/CameraBase.h`](../xrEngine/CameraBase.h.md)
**Used by** — reached through its declarations in [`actor_memory.h`](actor_memory.h.md); callers name that, not this file.
**Tier floor** — T2: a frustum and a visibility query per tracked object, on the AI budget

## Purpose

Creatures have a *feel* system that answers "what can I see"; the player needs the same
answer, because the game asks it constantly — a hint appears when the player looks at a
usable object, a creature reacts to being spotted, a script fires when the player first
sees something. Rather than a second mechanism, the player is given a vision client of its
own, wired to the player's camera instead of to a head bone.

The file is small because it makes exactly two decisions: **what the player's vision
frustum is** and **what is worth tracking in it**.

## State

Holds the player it belongs to. The tracked-object bookkeeping is the vision client's.

Invariant: the vision client is constructed with a re-check period of 100 milliseconds, so
the player's visibility set is refreshed ten times a second rather than every frame. That
is a deliberate cost decision: full-frame vision for the player would mean a raycast per
candidate per frame, and the consumers — hints, script callbacks, creature reactions — are
all tolerant of a tenth of a second.

## `camera`

**Contract** — supplies the vision frustum. Takes the player's *active* camera — which is
whichever of first person, look-at, free-look or fixed look-at is current — and reports
its position, direction, up vector, field of view and aspect ratio. The near plane is a
fixed tenth of a metre; the far plane is the weather system's current view distance.

**Invariants**

- Using the active camera, not the player's eyes, means that in a third-person or
  free-look mode the player "sees" what the *camera* sees. That is the correct behaviour
  for hints and for script triggers, and arguably the wrong one for whether a creature
  believes it has been spotted. The engine accepts the conflation.
- Tying the far plane to the current weather's view distance means the player's vision
  range shrinks in fog. This is a gameplay property, not a rendering one: in fog the
  player genuinely cannot be considered to have seen a distant creature, and creatures'
  own vision is bounded the same way.
- The field of view is converted from the camera's degrees to the vision system's
  radians here. The camera and the vision system disagree on units, and this is the one
  place the conversion happens.

```text
FUNCTION vision_frustum() -> (position, direction, up, fov, aspect, near, far)
  camera = player.active_camera
  position, direction, up = camera.transform
  fov    = radians(camera.field_of_view)
  aspect = camera.aspect_ratio
  near   = 0.1
  far    = current_weather.view_distance
```

## `feel_vision_isRelevant`

**Contract** — the filter deciding which objects the player's vision even considers.
Exactly one rule: only living entities. Items, doors, corpses' containers, anomalies and
level geometry are never tracked.

**Invariants** — this bounds the cost of player vision to the number of creatures nearby
rather than to the number of objects, which is two orders of magnitude smaller. The
consequence is that "what is the player looking at" for the purpose of a *usable-object
hint* is answered somewhere else entirely, by a ray cast from the crosshair — the two
questions look alike and are implemented separately for this reason.

**Notes** — the filter admits every living entity, including dead ones that still carry
the alive-entity interface, and including the player themselves in a networked game.
Consumers do their own further filtering.
