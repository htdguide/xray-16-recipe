# src/xrGame/map_spot.h

> Declares the four widget kinds a map marker draws itself with.

**Needs** — [`map_spot.cpp`](map_spot.cpp.md) · [`map_location.h`](map_location.h.md) · [`xrUICore/Static/UIStatic.h`](../xrUICore/Static/UIStatic.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md)
**Used by** — [`GameTask.cpp`](GameTask.cpp.md) · [`map_location.cpp`](map_location.cpp.md) · [`map_location.h`](map_location.h.md) · [`map_spot.cpp`](map_spot.cpp.md) · [`UIMap.cpp`](ui/UIMap.cpp.md) · [`UIMapWnd.cpp`](ui/UIMapWnd.cpp.md) · [`map_hint.cpp`](ui/map_hint.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the widget family behind a map marker. See [`map_spot.cpp`](map_spot.cpp.md).

Exported units:

- `CMapSpot` — the base: a picture that knows which marker it belongs to, whether it scales
  with map zoom and between what bounds, which map detail level it appears at, and an optional
  border decoration.
- `CMapSpot::Load` · `GetHint` · `Update` · `OnMouseDown` · `OnFocusLost` ·
  `show_static_border` · `mark_focused` · `SetWndPos`.
- `CMapSpotPointer` — the edge-of-map arrow; identical to the base except that it has no
  tooltip.
- `CMiniMapSpot` — a spot with three icon variants selected by the subject's height relative
  to the viewer.
- `CComplexMapSpot` — the rich level-map spot: three satellite icons and a countdown, all of
  which scale with the spot.
- `CUIStaticOrig` — a picture that remembers its authored position and size so it can be
  rescaled repeatedly without drift.

**Notes** — the height-band texture rectangles and shaders in the minimap spot are stored
per variant rather than swapped on a shared item, because the shared item's rectangle is
reused by the base widget every frame.
