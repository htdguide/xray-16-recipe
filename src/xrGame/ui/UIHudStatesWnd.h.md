# src/xrGame/ui/UIHudStatesWnd.h

> Declares the heads-up panel, its per-influence-type arrays, and the fake-indicator override.

**Needs** — [`UIHudStatesWnd.cpp`](UIHudStatesWnd.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrServerEntities/alife_space.h`](../../xrServerEntities/alife_space.h.md) · [`xrServerEntities/inventory_space.h`](../../xrServerEntities/inventory_space.h.md) · [`actor_defs.h`](../actor_defs.h.md)
**Used by** — [`UIGameCustom.cpp`](../UIGameCustom.cpp.md) · [`UIActorStateInfo.cpp`](UIActorStateInfo.cpp.md) · [`UIHudStatesWnd.cpp`](UIHudStatesWnd.cpp.md) · [`UIMainIngameWnd.cpp`](UIMainIngameWnd.cpp.md) · [`UIMainIngameWnd.h`](UIMainIngameWnd.h.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIHudStatesWnd.cpp`](UIHudStatesWnd.cpp.md).

One constant is declared here and is load-bearing: **the number of widgeted hazard types is
one fewer than the number of influence types**. Electric hazard is tracked and protected
against but has no indicator, and every loop over indicators uses the smaller bound while
every array is sized by the larger. A rebuild that unifies the two bounds either drops
electric hazard from the model or faults on a missing widget.

## Exported units

- **`CUIHudStatesWnd`**
  - `InitFromXml` — build from a layout; nearly every element optional.
  - `Load_section` / `on_connected` — read the per-hazard-type detector configuration —
    feel radius and threshold — and create the level's proximity list of hazard zones. Called
    on connecting to a level, because the configuration is per-game and the list per-level.
  - `Update` — vitals, weapon, indicators, zone sweep, in that order.
  - `reset_ui` — clear the proximity list; used on level change.
  - `UpdateHealth`, `UpdateActiveItemInfo`, `UpdateZones`, `UpdateIndicators` — the four
    subsystems, public so the game layer can drive them selectively.
  - `SetAmmoIcon` — size and position the weapon icon from a configuration section, with the
    per-game scale table.
  - `DrawZoneIndicators` — update and draw only the hazard triangles, for when the rest of
    the overlay is hidden.
  - `get_zone_cur_power(hit kind)` — the accumulated hazard for a hit kind, or zero for a
    hit kind with no indicator. This is read by the game layer, which is why it is public.
  - `get_main_sensor_value` — the needle's value, i.e. accumulated radiation.
  - `FakeUpdateIndicatorType` / `EnableFakeIndicators` — the script-driven override.
  - `get_indik_type` — the frozen many-to-one map from hit kind to influence type.

**Notes** — the commented-out fields record an abandoned design in which each hazard had
three discrete level pictures instead of one four-colour indicator, and the same for bleeding.
The surviving bleeding widget is the single-icon remnant of that.
