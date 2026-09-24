# src/xrGame/UIZoneMap.h

> Declares the minimap implemented in [`UIZoneMap.cpp`](UIZoneMap.cpp.md).

**Needs** — [`xrUICore/Static/UIStatic.h`](../xrUICore/Static/UIStatic.h.md)
**Used by** — [`UIZoneMap.cpp`](UIZoneMap.cpp.md) · [`UIMainIngameWnd.cpp`](ui/UIMainIngameWnd.cpp.md) · [`UIMainIngameWnd.h`](ui/UIMainIngameWnd.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CUIZoneMap`. Substance is in [`UIZoneMap.cpp`](UIZoneMap.cpp.md).

Note it is a plain object, not a window: it owns a widget tree but is not part of one,
so the heads-up layer must call its update and render explicitly.

Exported units:

- `Init(next_to_motion_icon)` — build the tree; the flag selects the square-in-screen-
  space layout with fraction-positioned decorations.
- `Render()` / `Update()` — draw clip frame then background; track camera position and
  heading.
- `SetupCurrentMap()` — bind to the loaded level's map texture and compute zoom.
- `OnSectorChanged(sector)` — swap to another vertical layer's map image.
- `Counter_ResetClrAnimation()` — restart the contacts counter's attention flash.
- `Background()` / `MapFrame()` — expose the two widgets the heads-up layer parents other
  indicators onto.
- `ZoomIn()` / `ZoomOut()` — accepted and ignored; the feature was cut.
