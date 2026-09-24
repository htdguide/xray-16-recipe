# src/xrEngine/Feel_Touch.h

> Declares the proximity-contact sense: the set of entities currently inside a radius, with enter and leave notifications.

**Needs** — [`pure_relcase.h`](pure_relcase.h.md)
**Used by** — [`Feel_Touch.cpp`](Feel_Touch.cpp.md) · [`BastArtifact.cpp`](../xrGame/BastArtifact.cpp.md) · [`BastArtifact.h`](../xrGame/BastArtifact.h.md) · [`BlackGraviArtifact.cpp`](../xrGame/BlackGraviArtifact.cpp.md) · [`BlackGraviArtifact.h`](../xrGame/BlackGraviArtifact.h.md) · [`CustomDetector.h`](../xrGame/CustomDetector.h.md) · [`CustomMonster.h`](../xrGame/CustomMonster.h.md) · [`CustomZone.cpp`](../xrGame/CustomZone.cpp.md) · [`CustomZone.h`](../xrGame/CustomZone.h.md) · [`Explosive.h`](../xrGame/Explosive.h.md) · [`GlobalFeelTouch.cpp`](../xrGame/GlobalFeelTouch.cpp.md) · [`GlobalFeelTouch.hpp`](../xrGame/GlobalFeelTouch.hpp.md) · [`Grenade.h`](../xrGame/Grenade.h.md) · [`PDA.cpp`](../xrGame/PDA.cpp.md) · _and 9 more_
**Tier floor** — T2.

## Purpose

Declares the surface implemented in [`Feel_Touch.cpp`](Feel_Touch.cpp.md).

## Exported units

- **`Touch`** — holds the current contact set, a scratch buffer for the per-update query, and a list of temporarily excluded entities. It is also a *release-case* participant, which is how it learns that an entity it holds is being destroyed.
- **`update`** — recompute the contact set around a point and a radius; the algorithm is in the implementation twin.
- **`contact`** — the per-candidate filter, overridden by the game to decide what counts as touching. The default accepts everything within the radius.
- **`on_enter` / `on_leave`** — the notifications. Both default to nothing.
- **`deny`** — exclude an entity from contact for a duration. Used so that an entity pushed out of a zone is not immediately re-detected.
- **`on_object_released`** — drop a destroyed entity, firing its leave notification.

**Notes** — The contact set is public and is read directly by game code, not only through the notifications. Both access patterns are load-bearing: the notifications drive state changes, and the set is walked each frame by anomalies applying continuous damage.
