# src/xrGame/ai/monsters/control_threaten.h

> Declares the threat-display ability and its payload, implemented in [`control_threaten.cpp`](control_threaten.cpp.md).

**Needs** — [`control_combase.h`](control_combase.h.md)
**Used by** — [`control_manager_custom.cpp`](control_manager_custom.cpp.md) · [`control_threaten.cpp`](control_threaten.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares `CControlThreaten`. Substance is in
[`control_threaten.cpp`](control_threaten.cpp.md).

## State

`SControlThreatenData` — the channel payload: a clip name and the fraction through the clip
at which the creature's threat-execute callback fires. Both are supplied by the creature at
load time through its custom manager.

Exported units: `reinit`, `update_schedule`, `activate`, `on_release`, `on_event`,
`check_start_conditions`. The scheduled update is what distinguishes this ability — it
keeps re-aiming at the enemy for the duration of the display.
