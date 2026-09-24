# src/xrGame/RadioactiveZone.h

> Declares the radiation anomaly implemented in [`RadioactiveZone.cpp`](RadioactiveZone.cpp.md).

**Needs** — [`CustomZone.h`](CustomZone.h.md)
**Used by** — [`RadioactiveZone.cpp`](RadioactiveZone.cpp.md) · [`mincer_script.cpp`](mincer_script.cpp.md) · [`space_restrictor.cpp`](space_restrictor.cpp.md) · [`stalker_anomaly_actions.cpp`](stalker_anomaly_actions.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CRadioactiveZone`, the generic zone specialized to continuous dose. Substance is
in [`RadioactiveZone.cpp`](RadioactiveZone.cpp.md).

Exported units:

- `CRadioactiveZone` — the anomaly.
- `Load` — reads the section; a hook only.
- `Affect` — the quantized dose integration.
- `feel_touch_new` — the zero-power entry hit that lights a client's indicator.
- `feel_touch_contact` — actors only, and the actor may refuse.
- `UpdateWorkload` — the server-side multiplayer dose path.
- `nearest_shape_radius` — the falloff radius for an object.
- `BlowoutState` — keeps the discharge animation running during the idle phase.
