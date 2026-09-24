# src/xrGame/MosquitoBald.h

> Declares the continuously damaging anomaly implemented in [`MosquitoBald.cpp`](MosquitoBald.cpp.md).

**Needs** — [`CustomZone.h`](CustomZone.h.md) · [`MosquitoBald.cpp`](MosquitoBald.cpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`MosquitoBald.cpp`](MosquitoBald.cpp.md) · [`MosquitoBald_script.cpp`](MosquitoBald_script.cpp.md) · [`TorridZone.cpp`](TorridZone.cpp.md) · [`TorridZone.h`](TorridZone.h.md) · [`ZoneCampfire.cpp`](ZoneCampfire.cpp.md) · [`ZoneCampfire.h`](ZoneCampfire.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CMosquitoBald`, the anomaly whose whole effect is damage — a strong hit at the
blowout and a weaker continuous one between them. Substance is in
[`MosquitoBald.cpp`](MosquitoBald.cpp.md).

Exported units:

- `CMosquitoBald` — the anomaly, carrying one flag that makes the blowout refresh run once per
  state transition rather than per tick.
- `Load` — a pure delegation to the base zone; the class introduces no tunables of its own.
- `Affect` — the blowout hit on one object.
- `BlowoutState` — the once-per-edge blowout refresh.
- `UpdateSecondaryHit` — the continuous between-blowouts damage.

The class registers itself with the script binding layer; its registration function also
exports two unrelated zone classes, which is documented in
[`MosquitoBald_script.cpp`](MosquitoBald_script.cpp.md).
