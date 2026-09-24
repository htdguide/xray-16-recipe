# src/xrGame/RegistryFuncs.h

> Declares the six accessors to the host's machine-wide settings store, implemented in [`RegistryFuncs.cpp`](RegistryFuncs.cpp.md).

**Needs** — _(none)_
**Used by** — [`RegistryFuncs.cpp`](RegistryFuncs.cpp.md) · [`console_commands_mp.cpp`](console_commands_mp.cpp.md) · [`login_manager.cpp`](login_manager.cpp.md) · [`UIOptConCom.cpp`](ui/UIOptConCom.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Names the read and write pair for each of the three value shapes the matchmaking layer
stores outside the game's own files. Substance is in
[`RegistryFuncs.cpp`](RegistryFuncs.cpp.md).

Exported units:

- `ReadRegistry_StrValue` / `WriteRegistry_StrValue` — a fixed 64-byte string.
- `ReadRegistry_DWValue` / `WriteRegistry_DWValue` — a 4-byte number.
- `ReadRegistry_BinaryValue` / `WriteRegistry_BinaryValue` — an opaque byte range with a
  caller-supplied capacity; the read returns the byte count actually retrieved.
