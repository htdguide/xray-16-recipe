# src/xrGame/actor_memory.h

> Declares the player's vision client, implemented in [`actor_memory.cpp`](actor_memory.cpp.md).

**Needs** — [`vision_client.h`](vision_client.h.md) · [`Actor.h`](Actor.h.md)
**Used by** — [`Actor.cpp`](Actor.cpp.md) · [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`CameraLook.cpp`](CameraLook.cpp.md) · [`WeaponBinocularsVision.cpp`](WeaponBinocularsVision.cpp.md) · [`actor_memory.cpp`](actor_memory.cpp.md) · [`psy_dog_aura.cpp`](ai/monsters/pseudodog/psy_dog_aura.cpp.md) · [`map_location.cpp`](map_location.cpp.md) · [`stalker_property_evaluators.cpp`](stalker_property_evaluators.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the player's participant in the senses system: a vision client bound to the
player, overriding the two questions the vision system asks of any client. Substance is in
[`actor_memory.cpp`](actor_memory.cpp.md).

Exported units:

- `feel_vision_isRelevant` — which objects are worth tracking (living entities only).
- `camera` — the frustum to test against (the player's active camera, far-clipped by the
  weather's view distance).

**Notes** — the name says *memory* but the type is vision only. The player has no
equivalent of a creature's remembered enemies, sounds or hits; what the player remembers
is the human's problem. A rebuild should call this what it is.
