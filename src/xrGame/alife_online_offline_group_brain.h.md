# src/xrGame/alife_online_offline_group_brain.h

> Declares the squad's offline brain, implemented in [`alife_online_offline_group_brain.cpp`](alife_online_offline_group_brain.cpp.md).

**Needs** — [`alife_space.h`](../xrServerEntities/alife_space.h.md) · [`xrServer_Space.h`](../xrServerEntities/xrServer_Space.h.md) · [`xrAICore/Navigation/game_graph_space.h`](../xrAICore/Navigation/game_graph_space.h.md) · [`alife_online_offline_group_brain_inline.h`](alife_online_offline_group_brain_inline.h.md)
**Used by** — [`alife_online_offline_group.cpp`](alife_online_offline_group.cpp.md) · [`alife_online_offline_group_brain.cpp`](alife_online_offline_group_brain.cpp.md) · [`alife_online_offline_group_brain_inline.h`](alife_online_offline_group_brain_inline.h.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CALifeOnlineOfflineGroupBrain`, which owns a movement manager on behalf of a
squad. Substance is in
[`alife_online_offline_group_brain.cpp`](alife_online_offline_group_brain.cpp.md).

The declared hook set is the *brain interface* the alife layer expects of any offline
decision loop — serialization in and out, registration and deregistration, a location
change, the online and offline transitions, and a per-tick update. It is not an abstract
base: the squad brain and the individual creature brain are separate concrete types that
happen to answer the same calls. A rebuild should make it a real interface, since that is
plainly what it is.

Exported units:

- `CALifeOnlineOfflineGroupBrain` — construct against a squad; creates the movement
  manager. Destruction releases it.
- `update` — the per-tick behaviour.
- `on_switch_online` / `on_switch_offline` — forwarded to the movement manager.
- `on_state_write`, `on_state_read`, `on_register`, `on_unregister`,
  `on_location_change` — all no-ops for a squad.
- `object`, `movement` — the squad and its movement manager.
