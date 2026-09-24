# src/xrGame/ui/UIMapWnd.h

> Declares the map screen: a world map with every level placed on it, a navigation cluster, a
> hint, a context menu, and the planner that animates the view from where it is to where it
> should be.

**Needs** — [`UIMapWnd.cpp`](UIMapWnd.cpp.md) · [`UIMapWnd2.cpp`](UIMapWnd2.cpp.md) · [`UIMapWndActions.h`](UIMapWndActions.h.md) · [`UIMap.h`](UIMap.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`GametaskManager.cpp`](../GametaskManager.cpp.md) · [`map_spot.cpp`](../map_spot.cpp.md) · [`UIMap.cpp`](UIMap.cpp.md) · [`UIMapWnd.cpp`](UIMapWnd.cpp.md) · [`UIMapWnd2.cpp`](UIMapWnd2.cpp.md) · [`UIMapWndActions.cpp`](UIMapWndActions.cpp.md) · [`UIPdaWnd.cpp`](UIPdaWnd.cpp.md) · [`UIPdaWnd.h`](UIPdaWnd.h.md) · [`UITaskWnd.cpp`](UITaskWnd.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented across
[`UIMapWnd.cpp`](UIMapWnd.cpp.md) (the screen) and
[`UIMapWnd2.cpp`](UIMapWnd2.cpp.md) (the navigation cluster). The split is arbitrary — one
class, two files — and a rebuild should merge them.

## Exported units

- **The map screen** — a window and a notification handler.
- `EBtnPos` — the nine navigation buttons in their authored order: legend, up, zoom in, left,
  centre on actor, right, zoom out, down, zoom reset. The order is the layout's grid order, so
  the index *is* the position; see the twin.
- `GAME_MAPS` — every level map, keyed by lowercased level name.
- `Init` — build from a named layout document; may decline when the document is absent.
- `SetTargetMap` in four forms — by map, by map and position, by name, by name and position;
  each optionally requesting a zoom-in. This is how everything outside asks the map to go
  somewhere.
- `ViewGlobalMap`, `ViewActor`, `ViewZoomIn`, `ViewZoomOut` — the four framed views.
- `MoveMap`, `MoveScrollV`, `MoveScrollH`, `UpdateScroll`, `SetZoom`, `GetZoom`, `UpdateZoom`
  — panning and zooming.
- `ShowHintStr`, `ShowHintSpot`, `ShowHintTask`, `HideHint`, `HideCurHint`, `DrawHint` — the
  single shared hint and its precedence rules.
- `SpotSelected`, `ActivatePropertiesBox` — what clicking a marker does.
- `AddMapToRender` / `RemoveMapToRender`, `GetMapByIdx`, `GetIdxByName`, `GameMaps`,
  `GlobalMap`, `ActiveMapRect` — the map registry and the visible area.
- `MapLocationRelcase` — drop references to a map location that is about to vanish.
- `m_tgtMap` / `m_tgtCenter` — the planner's goal, public because the planner reads it.
