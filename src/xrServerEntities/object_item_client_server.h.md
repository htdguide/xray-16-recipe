# src/xrServerEntities/object_item_client_server.h

> Declares the two paired registry entries: one that always builds the same class pair, and one that picks a different pair for single-player and for multiplayer.

**Needs** — [`object_item_abstract.h`](object_item_abstract.h.md) · [`object_factory.h`](object_factory.h.md) · [`object_factory_space.h`](object_factory_space.h.md)
**Used by** — [`object_factory_impl.h`](object_factory_impl.h.md) · [`object_item_client_server_inline.h`](object_item_client_server_inline.h.md)
**Tier floor** — T2.

## Purpose

Declares the surface implemented in
[`object_item_client_server_inline.h`](object_item_client_server_inline.h.md).

## Exported units

- **paired entry** — a registry entry bound to one client class and one server class.
  The overwhelming majority of entries.
- **switchable paired entry** — a registry entry bound to *four* classes: a client and a
  server for single player, and a client and a server for multiplayer. Chooses when it
  constructs, not when it is registered. Used for the actor, which carries different state
  in the two modes.
