# src/xrServerEntities/object_factory_space.h

> Names the two base types the registry deals in, so that the registry can be written without knowing what a client object or a server record is.

**Needs** — _(none; two forward declarations)_
**Used by** — [`object_factory.h`](object_factory.h.md) · [`object_item_abstract.h`](object_item_abstract.h.md) · [`object_item_client_server.h`](object_item_client_server.h.md) · [`object_item_single.h`](object_item_single.h.md)
**Tier floor** — T3.

## Purpose

The registry's whole surface is "construct a client object" and "construct a server record",
and this file is the one place those two words are bound to actual types: the client half is
the engine's generic constructible-object interface, the server half is the abstract server
record ([`xrServer_Object_Base.h`](xrServer_Object_Base.h.md)).

Its existence is the decision worth keeping: the registry, the entry shapes and the
registration list all depend on this file alone rather than on the whole entity hierarchy,
which is what lets the factory be compiled in the tools build where most gameplay classes
do not exist.

## State

`Stateless.`
