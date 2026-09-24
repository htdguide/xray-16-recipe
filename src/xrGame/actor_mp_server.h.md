# src/xrGame/actor_mp_server.h

> Declares the server-side networked player record, implemented across [`actor_mp_server.cpp`](actor_mp_server.cpp.md), [`actor_mp_server_export.cpp`](actor_mp_server_export.cpp.md) and [`actor_mp_server_import.cpp`](actor_mp_server_import.cpp.md).

**Needs** — [`xrServer_Objects_ALife_Monsters.h`](../xrServerEntities/xrServer_Objects_ALife_Monsters.h.md) · [`actor_mp_state.h`](actor_mp_state.h.md)
**Used by** — [`actor_mp_server.cpp`](actor_mp_server.cpp.md) · [`actor_mp_server_export.cpp`](actor_mp_server_export.cpp.md) · [`actor_mp_server_import.cpp`](actor_mp_server_import.cpp.md) · [`game_sv_mp.cpp`](game_sv_mp.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the networked player's *server object* — the authoritative record — as the alife
player record plus the compact wire-state holder and a freshness flag.

Exported units:

- `UPDATE_Read` / `UPDATE_Write` — the periodic compact update, in the same wire form the
  client object uses.
- `STATE_Read` / `STATE_Write` — the full record, used on first acquaintance.
- `Net_Relevant` — dead players are not relayed.
- `on_death` — capture the final state before relaying stops.

**Notes** — the same holder type appears in the client object and here, which is what
makes the server a pure relay: it does not translate between an internal representation
and a wire one, it holds the wire one.
