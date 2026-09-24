# src/xrGame/script_object.h

> Declares the plain scripted entity: a client object whose only behaviour is whatever the action queue tells it to do.

**Needs** — [`GameObject.h`](GameObject.h.md) · [`script_entity.h`](script_entity.h.md) · [`script_object.cpp`](script_object.cpp.md)
**Used by** — [`FryupZone.cpp`](FryupZone.cpp.md) · [`FryupZone.h`](FryupZone.h.md) · [`script_object.cpp`](script_object.cpp.md) · [`searchlight.cpp`](searchlight.cpp.md) · [`searchlight.h`](searchlight.h.md)
**Tier floor** — T2: a lifecycle composition

## Purpose

Declares the surface implemented in [`script_object.cpp`](script_object.cpp.md): the
simplest possible entity that a script can drive. It is a client object with the
[scripted-entity mixin](script_entity.h.md) attached and nothing else — no inventory, no
senses, no brain. Use it for a prop that must walk a path, play an animation and make a
sound on cue.

## Exported units

The full entity lifecycle, each member composing the two halves it inherits:

- construct, destroy.
- `construct_parts` — the deferred second half of construction the object factory calls.
- `reinit` — reset to the state a fresh spawn would have.
- `spawn_from(server_record)` — bring the entity online.
- `destroy_network_side` — take it offline.
- `uses_navigation_positions` — answers *no*; see the implementation twin.
- `scheduled_update(elapsed)` — the budgeted update.
- `frame_update` — the every-frame update.
- `as_script_entity` — identifies this object as carrying the scripted-entity mixin.
