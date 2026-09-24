# src/xrGame/cdkey_ban_list.h

> Declares the server's persistent ban list implemented in [`cdkey_ban_list.cpp`](cdkey_ban_list.cpp.md).

**Needs** — [`xrServer.h`](xrServer.h.md)
**Used by** — [`cdkey_ban_list.cpp`](cdkey_ban_list.cpp.md) · [`console_commands_mp.cpp`](console_commands_mp.cpp.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the surface implemented in
[`cdkey_ban_list.cpp`](cdkey_ban_list.cpp.md). The server owns one of these.

Exported units:

- **`load`, `save`** — read and rewrite the ban file.
- **`is_player_banned`** — the connection-time check, by product-key digest, reporting who
  issued the ban.
- **`ban_player`** — ban a connected client, refusing administrators and clients with no
  digest.
- **`ban_player_ll`** — ban by digest alone, for a player who is not connected.
- **`unban_player_by_index`** — remove one entry by its printed position.
- **`print_ban_list`** — the operator's listing, with a substring filter.
- **`banned_client`** (private) — one ban's record and its own load and save, described in
  the implementation twin.
