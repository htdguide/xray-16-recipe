# src/xrGame/team_base_zone.h

> Declares the multiplayer base capture zone.

**Needs** — [`team_base_zone.cpp`](team_base_zone.cpp.md) · [`GameObject.h`](GameObject.h.md) · [`xrEngine/Feel_Touch.h`](../xrEngine/Feel_Touch.h.md)
**Used by** — [`LevelGraphDebugRender.cpp`](LevelGraphDebugRender.cpp.md) · [`UIGameAHunt.cpp`](UIGameAHunt.cpp.md) · [`team_base_zone.cpp`](team_base_zone.cpp.md)
**Tier floor** — T2: a game object that also implements the touch sense.

## Purpose

Declares the surface implemented in [`team_base_zone.cpp`](team_base_zone.cpp.md).

The declaration's one decision is that a zone is a **game object that additionally
implements the touch sense**, rather than a game object that owns a sensor. Touch is one of
the three senses (`feel`) and is an opt-in interface: an object that wants to know what is
near it implements the interface and is then called back with entries and departures. That
is why the enter/leave operations below look like overrides rather than like callbacks
registered somewhere.

The class implements the standard entity lifecycle, and its order is load-bearing — see the
implementation: `reinit`, then `net_Spawn` from the authored record, then per-cycle
`shedule_Update`, then `net_Destroy`.

## Exported units

- `net_Spawn(record)` — build the shape set from the spawn record, take the owning team,
  compute bounds after placement, enable sensing, and add a map marker in multiplayer.
- `net_Destroy()` — remove the map marker and tear down.
- `reinit()` — pure delegation.
- `shedule_Update(dt)` — move the sense volume to the current transform and let the senses
  layer recompute who is inside.
- `feel_touch_contact(object)` — players only, tested against the shape set rather than the
  bounding sphere.
- `feel_touch_new(object)` / `feel_touch_delete(object)` — emit the enter and leave game
  events, on the authoritative side only.
- `Center(out)` / `Radius()` — the bounding sphere in world space.
- `GetZoneTeam()` — the owning team.
- in diagnostic builds, `OnRender()` — draw the shape set.
