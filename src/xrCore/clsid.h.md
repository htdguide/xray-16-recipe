# src/xrCore/clsid.h

> Declares the class identifier type and the compile-time packing, with the runtime conversions implemented in [`clsid.cpp`](clsid.cpp.md).

**Needs** — [`clsid.cpp`](clsid.cpp.md) · [`xr_types.h`](xr_types.h.md)
**Used by** — [`clsid.cpp`](clsid.cpp.md) · [`xrCore.h`](xrCore.h.md) · [`xr_ini.cpp`](xr_ini.cpp.md) · [`xr_ini.h`](xr_ini.h.md) · [`EngineAPI.h`](../xrEngine/EngineAPI.h.md) · [`xrGame.h`](../xrGame/xrGame.h.md) · [`clsid_game.h`](../xrServerEntities/clsid_game.h.md) · [`object_item_abstract.h`](../xrServerEntities/object_item_abstract.h.md)
**Tier floor** — T1: it fixes a 64-bit byte layout that appears in shipped data.

## Purpose

Declares the packed eight-character class identifier described in [`clsid.cpp`](clsid.cpp.md), and supplies the compile-time form of the packing so that an identifier written in source becomes a constant.

## Exported units

- **`CLASS_ID`** — an unsigned 64-bit integer holding eight characters, first character in the highest byte.
- **Compile-time packing** — from an eight-character array, or from eight separate characters. Both are constant expressions, so identifiers can label switch cases and initialize static tables.
- **Runtime conversions** — text to identifier (space-padded to eight) and identifier to text (nine-byte buffer, exact round trip). Contracts in [`clsid.cpp`](clsid.cpp.md).
