# src/xrGame/alife_registry_container.h

> Declares the bundle of per-character persistent registries and the type-keyed lookup that picks one out of it.

**Needs** — [`alife_registry_container_space.h`](alife_registry_container_space.h.md) · [`alife_registry_container_composition.h`](alife_registry_container_composition.h.md) · [`alife_abstract_registry.h`](alife_abstract_registry.h.md) · [`alife_registry_container_inline.h`](alife_registry_container_inline.h.md)
**Used by** — [`alife_registry_container.cpp`](alife_registry_container.cpp.md) · [`alife_registry_container_inline.h`](alife_registry_container_inline.h.md) · [`alife_registry_wrappers.h`](alife_registry_wrappers.h.md) · [`alife_simulator_base.cpp`](alife_simulator_base.cpp.md) · [`alife_simulator_script.cpp`](alife_simulator_script.cpp.md) · [`alife_storage_manager.cpp`](alife_storage_manager.cpp.md) · [`inventory_owner_info.cpp`](inventory_owner_info.cpp.md)
**Tier floor** — T3: a declaration, plus a lookup that a rebuild resolves at compile time or by name

## Purpose

Declares `CALifeRegistryContainer`: a single object that *is* every registry at once. The
container inherits from all of them, so it has one copy of each registry's storage, and a
caller selects one by naming its type rather than by naming a field.

The design decision worth carrying over is the **type-keyed selection**. Callers write
"give me the registry of relations" by mentioning the relation registry's type, not by
mentioning a field name or an index. That makes it impossible to reach a registry that is
not in the list — a compile-time error rather than a null result — and it means adding a
registry requires touching exactly one file, the composition list. A rebuild can
reproduce this with a type-indexed map, a generic accessor, or plain named fields; the
property to preserve is that *the list of registries has exactly one authoritative
definition*, because that list is also the save format's field order.

Save and load are overridable, because the container is constructed as part of the alife
simulator and the multiplayer server substitutes one that persists nothing.

Substance is in [`alife_registry_container.cpp`](alife_registry_container.cpp.md).

Exported units:

- `CALifeRegistryContainer` — the container; one instance per alife simulation.
- The type-keyed selector, readable and writable — see
  [`alife_registry_container_inline.h`](alife_registry_container_inline.h.md).
- `save` / `load` — the whole bundle as one chunk.

**Notes** — the multiple-inheritance chain that gives the container one member per
registry is generated from the type list by a library template. That is incidental. So is
the fact that the selector is spelled as a call taking a null pointer of the wanted type
— a C++ trick for passing a type as an argument, which a rebuild spells as a type
parameter, a name, or a field.
