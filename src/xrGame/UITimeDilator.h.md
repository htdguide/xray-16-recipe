# src/xrGame/UITimeDilator.h

> Declares the menu-time-dilation policy implemented in [`UITimeDilator.cpp`](UITimeDilator.cpp.md).

**Needs** — _(none beyond the core types)_
**Used by** — [`UIGameSP.cpp`](UIGameSP.cpp.md) · [`UIGameSP.h`](UIGameSP.h.md) · [`UITimeDilator.cpp`](UITimeDilator.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `UITimeDilator` and its two accessors. Substance is in
[`UITimeDilator.cpp`](UITimeDilator.cpp.md).

Exported units:

- `UIMode` — `None`, `Inventory`, `Pda`; bit values so the opt-in set is one flag word.
- `SetUiTimeFactor` / `GetUiTimeFactor` — the rate to run at while dilating.
- `SetModeEnability` / `GetModeEnability` — per-menu opt-in.
- `SetCurrentMode` — which menu is on top now; starts or stops dilation.
- `TimeDilator()` / `CloseTimeDilator()` — the lazily created process-wide instance.
