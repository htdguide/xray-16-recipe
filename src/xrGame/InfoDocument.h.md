# src/xrGame/InfoDocument.h

> Declares the pick-up document, implemented in [`InfoDocument.cpp`](InfoDocument.cpp.md).

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md) · [`InfoPortionDefs.h`](../xrServerEntities/InfoPortionDefs.h.md)
**Used by** — [`InfoDocument.cpp`](InfoDocument.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CInfoDocument`, an ordinary inventory item carrying one information-portion
identifier taken from its spawn record. Substance is in
[`InfoDocument.cpp`](InfoDocument.cpp.md).

Exported units:

- `CInfoDocument` — the item.
- `net_Spawn` — take the portion identifier from the server record.
- `OnH_A_Chield` — the one piece of behaviour: grant the portion to the new owner.
- `Load`, `net_Destroy`, `shedule_Update`, `UpdateCL`, `OnH_B_Independent` — delegation.
