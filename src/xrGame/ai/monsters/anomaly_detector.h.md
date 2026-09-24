# src/xrGame/ai/monsters/anomaly_detector.h

> Declares the component that makes a creature remember the anomalies it has bumped into and route around them for a while.

**Needs** — [`anomaly_detector.cpp`](anomaly_detector.cpp.md)
**Used by** — [`anomaly_detector.cpp`](anomaly_detector.cpp.md) · [`base_monster.cpp`](basemonster/base_monster.cpp.md) · [`base_monster_startup.cpp`](basemonster/base_monster_startup.cpp.md) · [`group_state_rest_inline.h`](group_states/group_state_rest_inline.h.md) · [`monster_state_rest_inline.h`](states/monster_state_rest_inline.h.md)
**Tier floor** — T3: a short list of (zone, timestamp) with an expiry sweep

## Purpose

Declares the surface implemented in [`anomaly_detector.cpp`](anomaly_detector.cpp.md).

## Exported units

- **the detector** — holds its creature, its detection radius, its memory period, an
  on/off switch, and the list of remembered anomalies.
- **remembered anomaly** — the zone and the moment it was registered; a registration time of
  zero means "noticed but not yet acted on", which is the entry's way of queueing itself for
  the next pass.
- **load, reinitialise, scheduled update, contact notification, enable, disable**.
