# src/xrGame/level_changer.h

> Declares the trigger volume that moves the player from one level to another.

**Needs** — [`GameObject.h`](GameObject.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md) · [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md)
**Used by** — [`actor_script.cpp`](actor_script.cpp.md) · [`level_changer.cpp`](level_changer.cpp.md) · [`map_location.cpp`](map_location.cpp.md) · [`script_game_object_script3.cpp`](script_game_object_script3.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CLevelChanger`, a game object that is also a *feel touch* sensor. See
[`level_changer.cpp`](level_changer.cpp.md) for the substance.

Exported units:

- `CLevelChanger` — the destination (game graph vertex, level vertex, position, angles),
  the enabled and silent flags, the prompt's string-table identifier, and the last time the
  prompt was raised.
- `net_Spawn` · `net_Destroy` — build the trigger shape from the server record; register and
  unregister in the global list of level changers.
- `Center` · `Radius` — the bounding sphere, taken from the collision shape.
- `shedule_Update` — refresh the touch set and re-offer the prompt.
- `feel_touch_new` · `feel_touch_contact` — who is inside, and what happens when the actor
  first enters.
- `IsVisibleForZones` — answers false: anomalies must not react to this object.
- `EnableLevelChanger` · `EnableSilentMode` · `SetLevelChangerInvitationStr` and their
  readers — the script-facing switches.
- `net_SaveRelevant` · `save` · `load` — persist the two things a script can change.
