# src/xrGame/ZoneCampfire.h

> Declares the switchable campfire zone implemented in [`ZoneCampfire.cpp`](ZoneCampfire.cpp.md).

**Needs** — [`MosquitoBald.h`](MosquitoBald.h.md)
**Used by** — [`MosquitoBald_script.cpp`](MosquitoBald_script.cpp.md) · [`ZoneCampfire.cpp`](ZoneCampfire.cpp.md) · [`script_game_object4.cpp`](script_game_object4.cpp.md) · [`script_game_object_script3.cpp`](script_game_object_script3.cpp.md) · [`space_restrictor.cpp`](space_restrictor.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CZoneCampfire` as an anomaly-zone subclass that can be extinguished and relit
from script, and names the state it adds: the two transitional particle emitters, the
extinguished-state sound, the target on/off flag and the transition deadline. Substance is
in [`ZoneCampfire.cpp`](ZoneCampfire.cpp.md).

Exported units:

- `turn_on_script` / `turn_off_script` / `is_on` — the script-facing switch.
- `Load`, `GoEnabledState`, `GoDisabledState` — the zone lifecycle hooks it overrides.
- `shedule_Update` — scheduled update; keeps the fade running while disabled and blows the
  smoke with the wind.
- `UpdateWorkload`, `PlayIdleParticles`, `StopIdleParticles`, `AlwaysTheCrow` — the
  protected hooks that implement the cross-fade.

**Notes** — the target-state flag defaults to *on*, so a campfire spawned from a level's
spawn record is burning unless a script puts it out.
