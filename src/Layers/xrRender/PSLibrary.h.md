# src/Layers/xrRender/PSLibrary.h

> Declares the single particle library — the name-keyed store of every effect and group definition — and the version and chunk identifiers of the library file.

**Needs** — [`PSLibrary.cpp`](PSLibrary.cpp.md) · [`ParticleEffectDef.h`](ParticleEffectDef.h.md) · [`ParticleGroup.h`](ParticleGroup.h.md) · [`Include/xrRender/particles_systems_library_interface.hpp`](../../Include/xrRender/particles_systems_library_interface.hpp.md)
**Used by** — [`particles_systems_library_interface.hpp`](../../Include/xrRender/particles_systems_library_interface.hpp.md) · [`PSLibrary.cpp`](PSLibrary.cpp.md) · [`ParticleGroup.cpp`](ParticleGroup.cpp.md)
**Tier floor** — T1: it fixes the library file's version word and container chunk identifiers.

## Purpose

Declares the surface implemented in [`PSLibrary.cpp`](PSLibrary.cpp.md). The file layout, the sort-and-binary-search invariant and the material lifetime rule are on that page.

Exported units:

- `ParticleLibrary` — the process's one store of effect and group definitions, and the implementor of the group-enumeration interface other modules reach it through.
- `on_create` / `on_destroy` / `reload` — bring the library up from game data and tear it down.
- `load` / `save` (packed library file) and `load_from_directory` / `save_to_directory` (loose configuration files).
- `find_effect` / `find_group` — exact-name lookup, and the cursor forms used by the editing surface.
- `append_effect` / `append_group` / `remove` / `rename_effect` / `rename_group` — the authoring surface; omittable in a play-only rebuild.
- `groups_begin` / `groups_end` / `groups_next` / `group_id` — the abstract group enumeration, whose end cursor is off by one (see the implementation twin).
- Format constants: library version 1; container identifiers for the second-generation effect list and the third-generation group list; a reserved first-generation identifier that is never read.

**Notes** — The header also defines a six-byte ASCII signature for the library file. Nothing reads or writes it anywhere in the engine, and the shipped library begins with a version chunk rather than a signature, so a rebuild should not emit one.
