# src/xrPhysics/PHInterpolation.h

> Declares the two-sample ring that lets the renderer draw between physics steps.

**Needs** — [`CycleConstStorage.h`](CycleConstStorage.h.md) · [`PHInterpolation.cpp`](PHInterpolation.cpp.md)
**Used by** — [`PHCharacter.h`](PHCharacter.h.md) · [`PHDynamicData.h`](PHDynamicData.h.md) · [`PHElement.cpp`](PHElement.cpp.md) · [`PHElement.h`](PHElement.h.md) · [`PHElementNetState.cpp`](PHElementNetState.cpp.md) · [`PHInterpolation.cpp`](PHInterpolation.cpp.md) · [`PHSimpleCharacter.cpp`](PHSimpleCharacter.cpp.md) · [`Physics.h`](Physics.h.md)
**Tier floor** — T2: two samples and a blend.

## Purpose

Declares the surface implemented in [`PHInterpolation.cpp`](PHInterpolation.cpp.md). The simulation
runs at a fixed rate and the display does not; this type is the bridge.

## Exported units

- `CPHInterpolation` — holds a body's last two solved placements and blends between them.
  - `set_body(body)` — bind and seed both samples from the body's current state.
  - `update_position` / `update_rotation` — push the current state as the newest sample.
  - `reset_positions` / `reset_rotations` — collapse both samples onto the current state, so the
    next blend produces no motion. Used after a teleport.
  - `interpolate_position(OUT point)` / `interpolate_rotation(OUT frame)` — the blend.
  - `get_position` / `get_rotation` / `set_position` / `set_rotation` by sample index — direct
    access to the two samples, used by the network state path, which must restore *both* so a
    corrected body does not visibly snap.
- `PH_INTERPOLATION_POINTS = 2` — the ring depth, and the reason the rest of the type is this
  simple.

## Notes

Two samples, not three: the engine interpolates between the previous and current solved states and
never extrapolates ahead. That means what is drawn always lags the simulation by up to one step —
about ten milliseconds — and never overshoots. Extrapolation would remove the lag at the cost of
visible correction snaps on every impact, which is much worse for objects that stop abruptly.
