# src/xrGame/ui/UIMapList.h

> Declares the map-rotation editor used when hosting a match: an available list, a chosen
> rotation, four transfer and reorder buttons, and the command line it all builds up to.

**Needs** — [`UIMapList.cpp`](UIMapList.cpp.md) · [`UIGameCustom.h`](../UIGameCustom.h.md) · [`gametype_chooser.h`](../../xrServerEntities/gametype_chooser.h.md) · [`xrUICore/Windows/UIWindow.h`](../../xrUICore/Windows/UIWindow.h.md)
**Used by** — [`ScriptXMLInit.cpp`](../ScriptXMLInit.cpp.md) · [`ChangeWeatherDialog.cpp`](ChangeWeatherDialog.cpp.md) · [`UIChangeMap.cpp`](UIChangeMap.cpp.md) · [`UIMapList.cpp`](UIMapList.cpp.md) · [`ui_export_script.cpp`](../ui_export_script.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIMapList.cpp`](UIMapList.cpp.md), plus the name of the
file the chosen rotation is written to.

## Exported units

- **The map-rotation editor** — a window owning two lists, two frames, two headers and four
  buttons.
- `InitFromXml` — build from a named subtree of the host screen's layout document.
- `SetWeatherSelector`, `SetModeSelector`, `SetMapPic`, `SetMapInfo`, `SetServerParams` — five
  injections of widgets and text the editor does *not* own but does read.
- `LoadMapList` / `SaveMapList` — fill the weather choices; write the rotation out.
- `GetCommandLine` — assemble the host-and-join command from the current choices.
- `StartDedicatedServer` — arrange for a headless server to be launched after this process
  quits, and quit.
- `GetCurGameType`, `IsEmpty`, `ClearList`, `GetMapNameInt`, `OnModeChange`,
  `OnListItemClicked` — the queries and the two externally driven refreshes.
- `MAP_ROTATION_LIST` — the rotation file's name, under the writable data root.
