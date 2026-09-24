# src/xrGame/map_manager.h

> Declares the owner of every map marker: the registry, the lookups, and the per-frame sweep.

**Needs** — [`map_manager.cpp`](map_manager.cpp.md) · [`map_location_defs.h`](map_location_defs.h.md) · [`map_location.h`](map_location.h.md)
**Used by** — [`Actor_Network.cpp`](Actor_Network.cpp.md) · [`Actor_Weapon.cpp`](Actor_Weapon.cpp.md) · [`GameTask.cpp`](GameTask.cpp.md) · [`GametaskManager.cpp`](GametaskManager.cpp.md) · [`Level.cpp`](Level.cpp.md) · [`Spectator.cpp`](Spectator.cpp.md) · [`actor_communication.cpp`](actor_communication.cpp.md) · [`alife_dynamic_object.cpp`](alife_dynamic_object.cpp.md) · [`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md) · [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md) · [`game_cl_capture_the_artefact_messages_menu.cpp`](game_cl_capture_the_artefact_messages_menu.cpp.md) · [`game_cl_deathmatch.cpp`](game_cl_deathmatch.cpp.md) · [`game_cl_mp_messages_menu.cpp`](game_cl_mp_messages_menu.cpp.md) · [`game_cl_teamdeathmatch.cpp`](game_cl_teamdeathmatch.cpp.md) · _and 8 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CMapManager`. See [`map_manager.cpp`](map_manager.cpp.md).

Exported units:

- `CMapManager` — the marker list (held indirectly through the save registry), the deferred
  destruction queue, and the shared map-spot description document.
- `Update` — advance every marker, reorder, and reap the stale ones.
- `AddMapLocation` · `AddRelationLocation` — create a marker of a named spot type, or the
  relation marker for one character.
- `RemoveMapLocation` (by spot type and entity, or by marker) · `RemoveMapLocationByObjectID`
  · `OnObjectDestroyNotify` — removal.
- `GetMapLocation` · `GetMapLocations` · `HasMapLocation` · `GetMapLocationsForObject` — the
  lookups.
- `Locations` — the marker list, resolved lazily.
- `DisableAllPointers` — clear every marker's pointer.
- `Destroy` — hand a marker to the deferred queue.
- `OnUIReset` — reload the spot descriptions and rebuild every marker's widgets.
- `ResetStorage` — forget the resolved list, so it is re-resolved after a save is loaded.
- `m_uiSpotXml` — the parsed map-spot description document, shared by every marker.
- `Dump` — a development-only listing.

**Notes** — the spot description document is a single shared instance rather than a per-marker
parse. Every marker navigates into it by name at load, and it is reloaded whenever the user
interface is reset, which is what makes a resolution change reload the map skin.
