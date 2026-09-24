# src/xrGame/HitMarker.h

> Declares the directional damage and grenade indicators, implemented in [`HitMarker.cpp`](HitMarker.cpp.md).

**Needs** — [`Include/xrRender/FactoryPtr.h`](../Include/xrRender/FactoryPtr.h.md) · [`xrUICore/ui_defs.h`](../xrUICore/ui_defs.h.md) · [`Grenade.h`](Grenade.h.md)
**Used by** — [`HUDManager.cpp`](HUDManager.cpp.md) · [`HUDManager.h`](HUDManager.h.md) · [`HitMarker.cpp`](HitMarker.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the two marker records and the collection that owns them. Substance is in
[`HitMarker.cpp`](HitMarker.cpp.md).

Exported units:

- `SHitMark` — one transient damage indicator: its sprite, the time it started, the
  world bearing it points at, and the shared fade curve. `IsActive` and `Draw`.
- `SGrenadeMark` — one persistent grenade warning: the tracked grenade, a detached flag,
  its sprite, the time it was last refreshed, its bearing and the same curve sampled at
  double rate. `IsActive`, `Update`, `Draw`.
- `CHitMarker` — the two expiring queues and the two materials.
  - `Hit` — record a hit, reversing the supplied travel direction into a bearing.
  - `AddGrenade_ForMark` — begin tracking a grenade, refusing a duplicate and reporting
    whether it is new.
  - `Update_GrenadeView` — refresh every tracked grenade's bearing, or detach it.
  - `Render` — retire expired markers and draw the rest, rotated into screen space.
  - `InitShader`, `InitShader_Grenade` — bind or re-bind the two textures.
  - `net_Relcase` — detach markers referencing an object about to be destroyed.

## Notes

Both marker records are declared with a destructor that releases their sprite, and the
collection deletes them explicitly. That ownership is incidental; what a rebuild must keep
is that a marker's sprite is per-marker rather than shared, because each is drawn at its
own rotation and alpha.
