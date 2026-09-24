# src/xrGame/actor_mp_client.h

> Declares the multiplayer player object, implemented across [`actor_mp_client.cpp`](actor_mp_client.cpp.md), [`actor_mp_client_export.cpp`](actor_mp_client_export.cpp.md) and [`actor_mp_client_import.cpp`](actor_mp_client_import.cpp.md).

**Needs** — [`Actor.h`](Actor.h.md) · [`actor_mp_state.h`](actor_mp_state.h.md) · [`anticheat_dumpable_object.h`](anticheat_dumpable_object.h.md)
**Used by** — [`actor_mp_client.cpp`](actor_mp_client.cpp.md) · [`actor_mp_client_export.cpp`](actor_mp_client_export.cpp.md) · [`actor_mp_client_import.cpp`](actor_mp_client_import.cpp.md) · [`configs_dumper.cpp`](configs_dumper.cpp.md) · [`mpactor_dump_impl.cpp`](mpactor_dump_impl.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares the networked player: the single-player character plus a wire-state holder, plus
the interface that lets an anti-cheat pass dump the object's live tuning values for
comparison against the server's.

Exported units:

- `net_Export` / `net_Import` / `net_Relevant` — the network lifecycle; substance in the
  export and import twins.
- `OnEvent` — intercept the consumable-use event.
- `Die` — force health to zero.
- `cam_Set` — refuse non-first-person cameras outside a debug build.
- `On_SetEntity` / `On_LostEntity` — swap in spectator camera smoothing.
- `DumpActiveParams` and the anti-cheat section name — write this player's live parameters
  into a configuration-format snapshot under a fixed section name, so a server can compare
  a client's effective tuning against the authoritative values. The section name is frozen
  because the comparison is by name.

**Notes** — deriving from both the anti-cheat dump interface and the player character
means a networked player *is* an ordinary player everywhere the rest of the engine looks
at it. That is what allows a single-player-shaped codebase to run a match at all.
