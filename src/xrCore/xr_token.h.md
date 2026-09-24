# src/xrCore/xr_token.h

> Declares the name/number table entry and its two lookups, implemented in [`xr_token.cpp`](xr_token.cpp.md).

**Needs** — [`xr_token.cpp`](xr_token.cpp.md) · [`xr_types.h`](xr_types.h.md)
**Used by** — [`ETextureParams.cpp`](../Layers/xrRender/ETextureParams.cpp.md) · [`xrRender_console.cpp`](../Layers/xrRender/xrRender_console.cpp.md) · [`xr_ini.cpp`](xr_ini.cpp.md) · [`xr_token.cpp`](xr_token.cpp.md) · [`xr_trims.cpp`](xr_trims.cpp.md) · [`xr_trims.h`](xr_trims.h.md) · [`Device_mode.cpp`](../xrEngine/Device_mode.cpp.md) · [`EngineAPI.cpp`](../xrEngine/EngineAPI.cpp.md) · [`StringTable.h`](../xrEngine/StringTable/StringTable.h.md) · [`death_anims.cpp`](../xrGame/death_anims.cpp.md) · [`game_sv_mp.cpp`](../xrGame/game_sv_mp.cpp.md) · [`PropertiesListTypes.h`](../xrServerEntities/PropertiesListTypes.h.md) · [`ai_sounds.h`](../xrServerEntities/ai_sounds.h.md) · [`alife_space.cpp`](../xrServerEntities/alife_space.cpp.md) · _and 4 more_
**Tier floor** — T1: the entry's alignment is fixed explicitly because the tables are laid out as raw arrays and read on architectures that fault on misaligned pointer loads.

## Purpose

Declares the token table described in [`xr_token.cpp`](xr_token.cpp.md).

## Exported units

- **`xr_token`** — a name and a number. Default-constructs to no name and -1, which is also the terminator shape.
- **Name for a number** — empty text on a miss.
- **Number for a name** — case-insensitive; -1 on a miss.

## Notes

The entry is declared with the alignment of a pointer and the build asserts that its size is exactly two pointers. The reason is recorded in the source and is worth keeping: these tables are written as brace-initialized static arrays and indexed as flat memory, and on architectures that require aligned pointer loads an entry padded differently than expected faults. A rebuild whose arrays are self-describing deletes both the alignment and the assertion.

The table's terminator is an entry with no name, not a count. That means a table may not contain an entry with an empty name, and nothing enforces it.
