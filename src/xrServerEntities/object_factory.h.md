# src/xrServerEntities/object_factory.h

> Declares the class-identifier registry: the single table that turns a tag from a spawn record into a live server record and, where one exists, a live client object.

**Needs** — [`object_item_abstract.h`](object_item_abstract.h.md) · [`object_factory_spawner.h`](object_factory_spawner.h.md) · [`object_factory_space.h`](object_factory_space.h.md) · [`xrEngine/editor_base.h`](../xrEngine/editor_base.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`GameObject.cpp`](../xrGame/GameObject.cpp.md) · [`PHDestroyable.cpp`](../xrGame/PHDestroyable.cpp.md) · [`ai_space.cpp`](../xrGame/ai_space.cpp.md) · [`alife_simulator.cpp`](../xrGame/alife_simulator.cpp.md) · [`alife_simulator_base.cpp`](../xrGame/alife_simulator_base.cpp.md) · [`xrGame.cpp`](../xrGame/xrGame.cpp.md) · [`xrgame_dll_detach.cpp`](../xrGame/xrgame_dll_detach.cpp.md) · [`object_factory.cpp`](object_factory.cpp.md) · [`object_factory_impl.h`](object_factory_impl.h.md) · [`object_factory_inline.h`](object_factory_inline.h.md) · [`object_factory_script.cpp`](object_factory_script.cpp.md) · [`object_factory_spawner.cpp`](object_factory_spawner.cpp.md) · [`object_item_client_server.h`](object_item_client_server.h.md) · [`object_item_script.cpp`](object_item_script.cpp.md) · _and 3 more_
**Tier floor** — T2: a sorted lookup table of constructors; nothing here touches bytes or devices.

## Purpose

Declares the surface described in [`object_factory.cpp`](object_factory.cpp.md),
[`object_factory_inline.h`](object_factory_inline.h.md) (the lookup and insertion
algorithms), [`object_factory_impl.h`](object_factory_impl.h.md) (how a registration
chooses its entry shape), [`object_factory_register.cpp`](object_factory_register.cpp.md)
(the table's contents) and [`object_factory_script.cpp`](object_factory_script.cpp.md)
(the script-facing half).

## Exported units

- **the factory type** — owns the registry, is created lazily on first use and destroyed
  when the script engine resets.
- `client_object(class identifier)` — build the live, local instance for a tag.
- `server_object(class identifier, section)` — build the authoritative record for a tag,
  tuned by a configuration section; a variant answers `none` instead of failing.
- `script_clsid(class identifier)` — the tag's *index* in the sorted table, which is the
  number scripts see.
- `register_script_class(...)` — add an entry whose constructors are Lua functions.
- `register_script()` — publish the whole table to the script layer as a name-to-index
  enumeration.
- `init_spawn_data()` / the tool frame — the debug spawner, present only outside the
  shipping build.

## Notes

The registry is also an editor tool in non-shipping builds: it inherits the debug-overlay
tool interface so that the same table that drives spawning can drive a spawn browser. That
is a build-shape decision, not an entity-record one; the record side works identically
either way.
