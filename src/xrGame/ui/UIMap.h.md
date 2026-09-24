# src/xrGame/ui/UIMap.h

> Declares the four map surfaces: the common metres-to-canvas base, the world map of levels,
> one level's map, and the round heads-up minimap.

**Needs** — [`UIMap.cpp`](UIMap.cpp.md) · [`xrUICore/Static/UIStatic.h`](../../xrUICore/Static/UIStatic.h.md) · [`xrUICore/Callbacks/UIWndCallback.h`](../../xrUICore/Callbacks/UIWndCallback.h.md)
**Used by** — [`LevelFogOfWar.cpp`](../LevelFogOfWar.cpp.md) · [`UIZoneMap.cpp`](../UIZoneMap.cpp.md) · [`map_location.cpp`](../map_location.cpp.md) · [`UIMap.cpp`](UIMap.cpp.md) · [`UIMapWnd.cpp`](UIMapWnd.cpp.md) · [`UIMapWnd.h`](UIMapWnd.h.md) · [`UIMapWnd2.cpp`](UIMapWnd2.cpp.md) · [`UIMapWndActions.cpp`](UIMapWndActions.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMap.cpp`](UIMap.cpp.md). Four types, one base and
three specialisations, each differing in what its coordinate space *means*.

## Exported units

- **`CUICustomMap`** — the base: a picture with a bound rectangle in world metres, the
  conversions between metres and canvas units, a working area to clip against, a lock flag, a
  rounded flag, and the pointer-to-off-screen-thing geometry. Subclasses override the spot
  refresh and, in two cases, the conversion itself.
- **`CUIGlobalMap`** — the world map. Its bound rectangle is in *canvas units*, not metres, so
  its conversion is a pure scale; it owns the zoom range and the pan clamp, and every level map
  is a child positioned inside it.
- **`CUILevelMap`** — one level's map. Carries a second rectangle giving its place on the world
  map, keeps its own rectangle derived from that every frame, and hosts the level's map spots.
- **`CUIMiniMap`** — the heads-up map. Rounded by default, which changes how it draws, how it
  decides a spot is visible, and where it puts an off-screen pointer.
- The working area, the bound rectangle, the zoom read-out, `FitToWidth` / `FitToHeight` /
  `OptimalFit`, `SetActivePoint`, `IsRectVisible`, `NeedShowPointer`, `GetPointerTo` and the
  lock and rounded flags are the shared surface every map answers.
