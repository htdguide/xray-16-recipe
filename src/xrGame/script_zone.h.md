# src/xrGame/script_zone.h

> Declares the scripted trigger volume: a restrictor that fires enter and exit callbacks into Lua.

**Needs** — [`space_restrictor.h`](space_restrictor.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md)
**Used by** — [`script_game_object2.cpp`](script_game_object2.cpp.md) · [`script_zone.cpp`](script_zone.cpp.md) · [`script_zone_script.cpp`](script_zone_script.cpp.md)
**Tier floor** — T2: an entity class on the scheduler

## Purpose

Declares the surface implemented in [`script_zone.cpp`](script_zone.cpp.md) and registered
to script in [`script_zone_script.cpp`](script_zone_script.cpp.md).

## Exported units

- **The class** — a space restrictor that also implements the *touch* sense, so it knows
  which client objects are inside its shape.
- **The lifecycle overrides** — reinitialize, spawn from a server record, destroy,
  reference release, scheduled update.
- **The three touch hooks** — object entered, object left, and the containment test that
  decides both.
- **`active_contact`** — is an entity, named by identifier, currently inside.
- **Two policy answers** — it is *not* visible to other zones (so zones do not trigger
  each other), and it *does* register with the scheduler (so its touch set updates even
  with nobody near).
- **A debug render override** — draws the restrictor's shapes.
