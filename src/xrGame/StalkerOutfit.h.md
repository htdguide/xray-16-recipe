# src/xrGame/StalkerOutfit.h

> Declares the stalker's suit, whose only source is its script registration in [`StalkerOutfit.cpp`](StalkerOutfit.cpp.md).

**Needs** — [`CustomOutfit.h`](CustomOutfit.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`StalkerOutfit.cpp`](StalkerOutfit.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CStalkerOutfit` as a generic outfit with no added state and no added behaviour,
plus the script registration entry point implemented in
[`StalkerOutfit.cpp`](StalkerOutfit.cpp.md).

Exported units:

- `CStalkerOutfit` — the outfit; everything it does comes from its configuration section.
- `script_register` — declares the type to Lua.
