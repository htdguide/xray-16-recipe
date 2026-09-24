# src/xrGame/server_entity_wrapper.h

> Declares a serializable envelope that lets one server object be written to and read from a stream on its own.

**Needs** — [`Common/object_interfaces.h`](../Common/object_interfaces.h.md) · [`xrServerEntities/xrServer_Objects.h`](../xrServerEntities/xrServer_Objects.h.md)
**Used by** — [`alife_spawn_registry.cpp`](alife_spawn_registry.cpp.md) · [`alife_spawn_registry.h`](alife_spawn_registry.h.md) · [`server_entity_wrapper.cpp`](server_entity_wrapper.cpp.md) · [`server_entity_wrapper_inline.h`](server_entity_wrapper_inline.h.md)
**Tier floor** — T2: stream framing around an existing record

## Purpose

Declares the surface implemented in
[`server_entity_wrapper.cpp`](server_entity_wrapper.cpp.md) and
[`server_entity_wrapper_inline.h`](server_entity_wrapper_inline.h.md).

## Exported units

- **The class** — implements the engine's serializable interface over one owned server
  object.
- **`save` / `load`** — write and read the object as a two-chunk record.
- **`save_update` / `load_update`** — declared and empty; see the implementation twin.
- **`object()`** — the wrapped server object.
- **Construction from a server object; destruction destroys it.**

## Notes

Ownership is total and one-way: the wrapper destroys whatever it was handed. Anything that
constructs one is giving the object away.
