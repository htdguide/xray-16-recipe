# src/xrGame/fire_disp_controller.h

> Declares the crosshair dispersion smoother, implemented in [`fire_disp_controller.cpp`](fire_disp_controller.cpp.md).

**Needs** — _(none)_
**Used by** — [`Actor.h`](Actor.h.md) · [`fire_disp_controller.cpp`](fire_disp_controller.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the object that eases the displayed crosshair spread toward the weapon's actual
dispersion. Substance is in
[`fire_disp_controller.cpp`](fire_disp_controller.cpp.md).

Exported units:

- `CFireDispertionController` — the smoother: a transition described by its two endpoints
  and a start time, plus the value currently on screen.
- `SetDispertion` — set a new target, starting a transition from the displayed value.
- `GetCurrentDispertion` — the displayed value.
- `Update` — advance the transition against the wall clock.
- `default_inertion` — the fallback rate in seconds per unit of dispersion, used when no
  weapon supplies one.
