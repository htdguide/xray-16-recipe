# src/xrGame/ui/UIMotionIcon.h

> Declares the stealth indicator: the actor's movement posture, stamina, emitted noise, and how
> visible the actor currently is to anyone watching.

**Needs** — [`UIMotionIcon.cpp`](UIMotionIcon.cpp.md) · [`xrUICore/ProgressBar/UIProgressBar.h`](../../xrUICore/ProgressBar/UIProgressBar.h.md) · [`xrUICore/ProgressBar/UIProgressShape.h`](../../xrUICore/ProgressBar/UIProgressShape.h.md)
**Used by** — [`UIMainIngameWnd.cpp`](UIMainIngameWnd.cpp.md) · [`UIMainIngameWnd.h`](UIMainIngameWnd.h.md) · [`UIMotionIcon.cpp`](UIMotionIcon.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMotionIcon.cpp`](UIMotionIcon.cpp.md).

## Exported units

- **The stealth indicator** — a picture with six mutually exclusive posture icons and up to
  three progress readouts.
- `EState` — the six postures: normal, crouch, creep, climb, run, sprint, plus a sentinel
  meaning *none shown yet*.
- `Init` — build from its own layout document; reports whether the document wants it parented
  into the minimap.
- `AttachToMinimap` — resize and centre inside the minimap frame.
- `ShowState`, `SetPower`, `SetNoise`, `SetLuminosity` — the four inputs.
- `SetActorVisibility` / `ResetVisibility` — per-observer visibility contributions and their
  reset.
