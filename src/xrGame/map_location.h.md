# src/xrGame/map_location.h

> Declares a map marker: one entity's presence on the level map, the minimap and the pointer ring, in whatever visual form its spot type describes.

**Needs** — [`map_location.cpp`](map_location.cpp.md) · [`map_spot.h`](map_spot.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md)
**Used by** — [`GameTask.cpp`](GameTask.cpp.md) · [`GametaskManager.cpp`](GametaskManager.cpp.md) · [`game_cl_artefacthunt.cpp`](game_cl_artefacthunt.cpp.md) · [`game_cl_capture_the_artefact.cpp`](game_cl_capture_the_artefact.cpp.md) · [`game_cl_capture_the_artefact_messages_menu.cpp`](game_cl_capture_the_artefact_messages_menu.cpp.md) · [`game_cl_mp_messages_menu.cpp`](game_cl_mp_messages_menu.cpp.md) · [`game_cl_teamdeathmatch.cpp`](game_cl_teamdeathmatch.cpp.md) · [`level_script.cpp`](level_script.cpp.md) · [`map_location.cpp`](map_location.cpp.md) · [`map_location_defs.h`](map_location_defs.h.md) · [`map_manager.cpp`](map_manager.cpp.md) · [`map_manager.h`](map_manager.h.md) · [`map_script.cpp`](map_script.cpp.md) · [`map_spot.cpp`](map_spot.cpp.md) · _and 7 more_
**Tier floor** — T3: a declaration

## Purpose

Declares `CMapLocation` and its one specialization, `CRelationMapLocation`. See
[`map_location.cpp`](map_location.cpp.md) for the substance.

Exported units:

- `ELocationFlags` — nine independent bits: serializable, hide while the subject is offline,
  has a time to live, position tracks the actor instead of the subject, pointer enabled, spot
  enabled, collidable, hint enabled, user defined.
- `CMapLocation` — the spot widgets (up to three kinds, each with an optional pointer and up
  to two border decorations), the subject's identifier and its authoritative record, the
  hint, the time-to-live, and a per-frame cache of the derived position, direction and level
  name.
- `LoadSpot` — build the widgets from the spot-type description.
- `Update` · `UpdateTTL` — advance the cache for this frame; report whether the marker is
  still meaningful.
- `UpdateMiniMap` · `UpdateLevelMap` — place the marker on one of the two maps.
- `CalcPosition` · `CalcDirection` · `GetPosition` · `GetLastPosition` · `GetLevelName` —
  the derived geometry.
- `InitUserSpot` — pin a marker to a fixed world position instead of an entity.
- `SetHint` · `GetHint` · `HintEnabled` — the tooltip.
- `EnableSpot` · `DisableSpot` · `EnablePointer` · `DisablePointer` · `HighlightSpot` ·
  `SpotSize` · `Collidable` · `IsUserDefined` — presentation switches.
- `Serializable` · `SetSerializable` · `save` · `load` — persistence.
- `CRelationMapLocation` — a marker whose appearance and visibility follow the faction
  relation between its subject and the player.

**Notes** — the twelve widget fields are the same three-way split repeated four times: level
map, minimap, complex spot; each with a spot, a pointer, an active border and an inactive
border. A rebuild collapses them into three records of four fields and loses nothing.
