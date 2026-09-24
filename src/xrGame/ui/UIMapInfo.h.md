# src/xrGame/ui/UIMapInfo.h

> Declares the multiplayer map description panel: a scrolling block of text assembled from one
> map's own description file.

**Needs** — [`UIMapInfo.cpp`](UIMapInfo.cpp.md) · [`UIMapInfo_script.cpp`](UIMapInfo_script.cpp.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`ScriptXMLInit.cpp`](../ScriptXMLInit.cpp.md) · [`UIMapInfo.cpp`](UIMapInfo.cpp.md) · [`UIMapInfo_script.cpp`](UIMapInfo_script.cpp.md) · [`UIMapList.cpp`](UIMapList.cpp.md) · [`UIServerInfo.cpp`](UIServerInfo.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMapInfo.cpp`](UIMapInfo.cpp.md), and marks the type as
one that registers itself with the script layer — see
[`UIMapInfo_script.cpp`](UIMapInfo_script.cpp.md).

## Exported units

- **The map info panel** — a window wrapping one scroll view.
- `InitMapInfo` — place and size the panel and bring up its scroll view.
- `InitMap` — rebuild the panel's content for a named map and version.
- `GetLargeDesc` — the long description, if the map's file carried one. Set as a side effect
  of the rebuild; empty otherwise.
