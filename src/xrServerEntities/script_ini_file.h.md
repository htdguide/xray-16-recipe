# src/xrServerEntities/script_ini_file.h

> Declares the configuration file as scripts see it: the engine's parser with every read turned into a diagnosable failure, plus a write surface the engine itself does not use.

**Needs** — [`script_token_list.h`](script_token_list.h.md) · [`xrCore/xr_ini.h`](../xrCore/xr_ini.h.md) · [Data: configuration](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`script_game_object_script.cpp`](../xrGame/script_game_object_script.cpp.md) · [`script_game_object_script3.cpp`](../xrGame/script_game_object_script3.cpp.md) · [`script_ini_file.cpp`](script_ini_file.cpp.md) · [`script_ini_file_script.cpp`](script_ini_file_script.cpp.md) · [`xrServer_Objects_script.cpp`](xrServer_Objects_script.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in [`script_ini_file.cpp`](script_ini_file.cpp.md) and
exported in [`script_ini_file_script.cpp`](script_ini_file_script.cpp.md).

## Exported units

- **script configuration file** — constructible from a path (resolved against the game's
  configuration root), from a path under an explicit logical root, or from an in-memory
  reader.
- the checked readers: string, unsigned, signed, float, three-component vector, line count,
  token, and class identifier.
- the writers: one per scalar type, plus save-under-a-new-name and remove-a-line.
- `resolve(root, name)` — the path resolution used by the constructors.
