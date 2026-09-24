# src/xrGame/ui/ServerList.h

> Declares the multiplayer server browser: a sortable, filtered list of servers with an
> expandable detail pane — reaching a matchmaking service that no longer exists.

**Needs** — [`ServerList.cpp`](ServerList.cpp.md) · [`UIListItemServer.h`](UIListItemServer.h.md) · [`UIMessageBoxEx.h`](UIMessageBoxEx.h.md) · [`../../xrUICore/ListBox/UIListBox.h`](../../xrUICore/ListBox/UIListBox.h.md) · [`../../xrUICore/Windows/UIFrameWindow.h`](../../xrUICore/Windows/UIFrameWindow.h.md) · [`../../xrUICore/EditBox/UIEditBox.h`](../../xrUICore/EditBox/UIEditBox.h.md) · [`../../xrUICore/Buttons/UI3tButton.h`](../../xrUICore/Buttons/UI3tButton.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`ScriptXMLInit.cpp`](../ScriptXMLInit.cpp.md) · [`ServerList.cpp`](ServerList.cpp.md) · [`ServerList_GameSpy_func.cpp`](ServerList_GameSpy_func.cpp.md) · [`ui_export_script.cpp`](../ui_export_script.cpp.md)
**Tier floor** — T2: a recycling widget pool over an external, asynchronous data source

## Purpose

Declares the surface implemented in [`ServerList.cpp`](ServerList.cpp.md).

**Read the seam first.** The vendor matchmaking service this browser queries was shut down and
nothing answers. Every path below is reachable and exercised; the list is permanently empty.
The browser is documented because it states, precisely, *what a replacement master server
would have to provide*.

## The three lists

`LST_SERVER`, `LST_SRV_PROP`, `LST_PLAYERS` — the server list, and the two detail lists that
slide into view beneath it. Constants, and the code addresses them by those names.

## `LST_COLUMN_COUNT` is seven

Icon, server name, map, game type, players, ping, version. Six of the seven are sortable
column headers; the icon column is not. The count is frozen by the layout document.

## `SServerFilters`

The client-side filter set: hide empty servers, hide full ones, hide password-protected ones,
hide unprotected ones, hide friendly-fire servers, hide listen (non-dedicated) servers. One
field is declared and never read by the engine — it exists because a game's shipped scripts
reference it, and removing it breaks them.

## `ESortingMode` / `ESortingType`

Six sort keys matching the six sortable columns, and three sort directions — ascending,
descending, and *auto*, which means "toggle if this is the same column, otherwise ascending".

## `CServerList`

The browser. Notable state: a **pool of recycled row widgets** rather than one widget per
server, a subscription to the browser service's update callback, a deferred-refresh frame
stamp, and a remembered selection that survives a refresh.

## `connect_error_cb`

A callback the owning screen installs to be told why a connection attempt was refused, with
two reasons — the account nickname is unregistered, or it has expired. Both are artefacts of
the dead account service.
