# src/xrGame/TorridZone.h

> Declares the moving burner anomaly implemented in [`TorridZone.cpp`](TorridZone.cpp.md).

**Needs** — [`MosquitoBald.h`](MosquitoBald.h.md)
**Used by** — [`MosquitoBald_script.cpp`](MosquitoBald_script.cpp.md) · [`TorridZone.cpp`](TorridZone.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CTorridZone`, the burner zone with an authored motion clip driving its
transform. Substance is in [`TorridZone.cpp`](TorridZone.cpp.md).

Exported units:

- `CTorridZone` — the anomaly; owns the motion clip for its whole life.
- `net_Spawn` — loads and starts the clip named in the server record.
- `UpdateWorkload` — advances the clip and moves the zone.
- `shedule_Update` — re-anchors the zone's four sounds.
- `Enable` / `Disable` — restart and stop the clip with the zone.
- `IsVisibleForZones` — true; this zone can be felt by other zones.
- `light_in_slow_mode` — false; the light must track the moving transform every frame.
- `AlwaysTheCrow` — true; always in the every-frame update set.

## Notes

The declaration names the generic zone, not the burner zone, as the parent to defer to.
See the note in [`TorridZone.cpp`](TorridZone.cpp.md).
