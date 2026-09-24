# src/xrServerEntities/object_item_script.h

> Declares the registry entry whose two constructors are script functions rather than engine classes.

**Needs** — [`object_item_abstract.h`](object_item_abstract.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`object_factory_script.cpp`](object_factory_script.cpp.md) · [`object_item_script.cpp`](object_item_script.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in
[`object_item_script.cpp`](object_item_script.cpp.md).

## Exported units

- **script-backed entry** — holds two callable script values, one per half, and a
  constructor that binds the same value to both when a single script class plays both.
  Construction transfers ownership of the produced object from the script layer to the
  engine, so the script garbage collector will not free something the world is holding.
