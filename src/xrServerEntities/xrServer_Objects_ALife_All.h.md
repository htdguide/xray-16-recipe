# src/xrServerEntities/xrServer_Objects_ALife_All.h

> One include that pulls in every concrete entity record.

**Needs** — [`xrServer_Objects_ALife_Items.h`](xrServer_Objects_ALife_Items.h.md) · [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md)
**Used by** — [`Level_network_spawn.cpp`](../xrGame/Level_network_spawn.cpp.md) · [`alife_graph_registry.cpp`](../xrGame/alife_graph_registry.cpp.md) · [`alife_graph_registry.h`](../xrGame/alife_graph_registry.h.md) · [`xrServer.cpp`](../xrGame/xrServer.cpp.md) · [`object_factory_register.cpp`](object_factory_register.cpp.md)
**Tier floor** — T4: an aggregation.

## Purpose

A convenience umbrella for the two files that between them declare every concrete record:
items and monsters. The split is arbitrary — items and monsters share most of their bases —
and a rebuild is free to have one module or twenty. Nothing decides anything here.
