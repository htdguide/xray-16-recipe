# src/xrGame/ai/monsters/control_critical_wound.h

> Declares the critical-wound collapse and its payload, implemented in [`control_critical_wound.cpp`](control_critical_wound.cpp.md).

**Needs** — [`control_combase.h`](control_combase.h.md)
**Used by** — [`control_critical_wound.cpp`](control_critical_wound.cpp.md) · [`control_manager_custom.cpp`](control_manager_custom.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControlCriticalWound`. Substance is in
[`control_critical_wound.cpp`](control_critical_wound.cpp.md).

## State

`SControlCriticalWoundData` — the channel payload: one clip name.

Exported units: `activate`, `on_release`, `on_event`, `check_start_conditions`. The release
path, not the event, is where the creature is told the wound state has ended.
