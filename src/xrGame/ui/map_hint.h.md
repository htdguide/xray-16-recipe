# src/xrGame/ui/map_hint.h

> Declares the map's tooltip: one panel that renders either a plain line or a full task summary.

**Needs** — [`map_hint.cpp`](map_hint.cpp.md) · [`xrUICore/Windows/UIFrameWindow.h`](../../xrUICore/Windows/UIFrameWindow.h.md)
**Used by** — [`UIMapWnd.cpp`](UIMapWnd.cpp.md) · [`map_hint.cpp`](map_hint.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in [`map_hint.cpp`](map_hint.cpp.md).

Exported units:

- `CUIMapLocationHint` — the tooltip panel: a name-keyed table of labels, of which one subset or the
  other is shown.
- `Init(document, path)` — build, tolerating two layout vocabularies.
- `SetInfoStr(text)` — the plain mode.
- `SetInfoTask(task)` — the task mode.
- `SetInfoMSpot(spot)` — choose between them from a map spot.
- `SetOwner` / `GetOwner` — which widget the tooltip currently belongs to, so the map can tell when
  to dismiss it.
