# src/xrServerEntities/object_factory_impl.h

> Decides, at registration time, which shape of registry entry a class needs: a client/server pair, or a lone class that is one or the other.

**Needs** — [`object_factory.h`](object_factory.h.md) · [`object_item_single.h`](object_item_single.h.md) · [`object_item_client_server.h`](object_item_client_server.h.md) · [`Common/object_type_traits.h`](../Common/object_type_traits.h.md)
**Used by** — [`object_factory_register.cpp`](object_factory_register.cpp.md)
**Tier floor** — T2: a compile-time dispatch on which base a class derives from.

## Purpose

Registration is written two ways in
[`object_factory_register.cpp`](object_factory_register.cpp.md) — a pair (`client, server`)
or a single class — and this file turns each into the right entry shape while checking that
the classes actually are what the caller claims.

## `add` (paired form)

**Contract** — registers a tag whose entity has both halves: a client object (the live,
renderable, simulated instance) and a server object (the authoritative record). Refuses at
compile time if the client class does not descend from the client base or the server class
from the server base. Produces a paired entry.

## `add` (single form)

**Contract** — registers a tag whose entity has only one half, and works out *which* half
from the class's ancestry: a class descending from the server base becomes a server-only
entry, one descending from the client base becomes a client-only entry. Refuses if it is
neither.

**Notes** — the single form exists for two populations. Some things are pure records with no
live instance: the game-graph point, the monster group template, the online/offline group.
Others are pure live objects with no persisted record: the multiplayer game-rules objects
and the game screens. Asking the entry which half it has, and aborting loudly on the missing
one (see [`object_item_single_inline.h`](object_item_single_inline.h.md)), is how a
mis-registration surfaces immediately rather than as a null dereference later.

The "which base does it derive from" question is answered at compile time here. A rebuild
without that facility can carry the answer explicitly at each registration site — the
information is a property of the class, written down once either way.
