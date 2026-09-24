# src/xrServerEntities/object_item_abstract.h

> What a registry entry is: one class identifier, one script name, and the ability to construct either half of the entity.

**Needs** — [`object_factory_space.h`](object_factory_space.h.md) · [`xrCore/clsid.h`](../xrCore/clsid.h.md)
**Used by** — [`object_factory.h`](object_factory.h.md) · [`object_factory_inline.h`](object_factory_inline.h.md) · [`object_item_abstract_inline.h`](object_item_abstract_inline.h.md) · [`object_item_client_server.h`](object_item_client_server.h.md) · [`object_item_script.h`](object_item_script.h.md) · [`object_item_single.h`](object_item_single.h.md)
**Tier floor** — T2: an interface with two constructor operations.

## Purpose

This is the interface every registry entry satisfies, and therefore the contract a rebuild
must reproduce however it stores its table. There are four implementations — native pair,
native single, single-player/multiplayer switchable, and script-backed — and the registry
knows only this shape.

## State

```text
RECORD RegistryEntry
  identifier  : ClassIdentifier   # the frozen eight-character tag, as a 64-bit value
  script_name : text              # the key this class is published under to scripts
```

**Invariants** — both fields are set once at construction and never change. The identifier
is the sort key of the whole registry; the script name is its published alias. Both are
unique across the table.

## `identifier` / `script_name`

**Contract** — read-only accessors. No side effects.

## `client_object`

**Contract** — construct the live, local instance for this class and hand back ownership.
An entry with no client half fails loudly rather than answering nothing, because the caller
has already decided this tag is renderable and a silent nothing would surface much later.

## `server_object`

**Contract** — construct the authoritative record for this class, tuned by the named
configuration section, and hand back ownership. The section is required: an entity is
(class, section), and the record reads its defaults out of that section during
construction. An entry with no server half fails loudly.

**Invariants** — the returned record has run its post-construction initialization step
before it is handed over; see the implementations.
