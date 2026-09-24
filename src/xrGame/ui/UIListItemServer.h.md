# src/xrGame/ui/UIListItemServer.h

> Declares the server-browser row and the four records that describe a server to it.

**Needs** — [`UIListItemServer.cpp`](UIListItemServer.cpp.md) · [`xrUICore/ListBox/UIListBoxItem.h`](../../xrUICore/ListBox/UIListBoxItem.h.md) · [Seam: Multiplayer matchmaking and accounts](../../../SYSTEM-REQUIREMENTS.md#seam-multiplayer-matchmaking-and-accounts)
**Used by** — [`ServerList.cpp`](ServerList.cpp.md) · [`ServerList.h`](ServerList.h.md) · [`UIListItemServer.cpp`](UIListItemServer.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`UIListItemServer.cpp`](UIListItemServer.cpp.md). The
records are the substance: they are the **shape of a server as the browser knows it**, and
that shape is dictated by what the matchmaking service returned.

```text
RECORD ServerColumnWidths            # authored, in canvas units
  icon, server, map, game, players, ping, version, height : real

RECORD ServerFlags
  password_required : bool
  dedicated         : bool
  punkbuster        : bool          # no longer drawn
  account_required  : bool

RECORD ServerInfo
  server, address, map, game, players, ping, version : text
  icons : ServerFlags
  index : int                        # position in the service's result set

RECORD ServerRowParams
  text_colour : colour
  text_font   : Font
  size        : ServerColumnWidths
  info        : ServerInfo
```

**Notes** — the counts and latency arrive as **text**, not numbers, because the service
returned them formatted. A browser that wanted to sort by latency would have to parse them
back; the shipped one sorts on the service's side, which is why it does not.

## Exported units

- **`CUIListItemServer`**
  - constructed with the row height; builds three icon slots and six text columns.
  - `InitItemServer` — place everything from the widths and dress the icons.
  - `SetParams` — fill the text, show or hide the icons, set the tag.
  - `CreateConsoleCommand` — compose the join command from the row's address and the
    player's credentials.
  - `Get_gs_index` / `GetInfo` — the service index and the whole record.
