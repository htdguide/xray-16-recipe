# src/xrGame/game_sv_item_respawner.h

> Declares the multiplayer item respawner — the thing that keeps weapons and ammunition appearing at their pickup points — implemented in [`game_sv_item_respawner.cpp`](game_sv_item_respawner.cpp.md).

**Needs** — [`game_sv_item_respawner.cpp`](game_sv_item_respawner.cpp.md) · [`game_base.h`](game_base.h.md) · [`xrServerEntities/xrServer_Object_Base.h`](../xrServerEntities/xrServer_Object_Base.h.md)
**Used by** — [`game_sv_base.cpp`](game_sv_base.cpp.md) · [`game_sv_base.h`](game_sv_base.h.md) · [`game_sv_item_respawner.cpp`](game_sv_item_respawner.cpp.md)
**Tier floor** — T3: a declaration

## Purpose

Declares the server-side item respawner. Substance is in
[`game_sv_item_respawner.cpp`](game_sv_item_respawner.cpp.md).

The declaration decides one thing worth naming: there are **two independent item
populations** and the class handles both. The first is the pickup points authored into the
level's respawn table, each of which cycles on its own timer; the second is a whole spawn
file of loose level items that is wiped and reloaded at a round boundary. They share nothing
but this class.

Exported units:

- `item_respawn_manager` — the respawner.
- `spawn_item` (private) — one cycling pickup: the prototype entity, the respawn delay, the
  identifier of the instance currently out in the world, and when it was taken.
- `section_item` (private) — one row of the respawn configuration: what to spawn, after how
  long, with which attachments, and with how much ammunition.
- `add_new_rpoint` — register a pickup point with a named loadout profile. Called during
  level setup only.
- `respawn_all_items` — spawn every registered pickup at once.
- `respawn_level_items` — destroy and reload the loose level items.
- `check_to_delete` — report that an item was taken, starting its respawn timer.
- `update` — advance the timers.
- `clear_respawns` — drop every registered pickup and its prototype.
