# src/xrGame/alife_storage_manager_inline.h

> Construction of the save/load layer.

**Needs** — [`alife_storage_manager.h`](alife_storage_manager.h.md)
**Used by** — [`alife_storage_manager.h`](alife_storage_manager.h.md)
**Tier floor** — T3: two field initializers

## Purpose

One body, split out as a C++ habit. Substance is in
[`alife_storage_manager.cpp`](alife_storage_manager.cpp.md).

## Construction

**Contract** — records the configuration section the simulation is built from, and starts
with an empty current-save name.

**Invariants** — the section is stored rather than merely forwarded, because loading a
save destroys the whole simulation and must rebuild it from the same section. An empty
initial save name means the first save must be given a name explicitly; only afterwards
does the reuse-the-last-name behaviour become available.
