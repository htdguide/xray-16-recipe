# src/xrServerEntities/object_item_single.h

> Declares the registry entry for a class that has only one half — a record with no live object, or a live object with no record.

**Needs** — [`object_item_abstract.h`](object_item_abstract.h.md) · [`object_factory_space.h`](object_factory_space.h.md)
**Used by** — [`object_factory_impl.h`](object_factory_impl.h.md) · [`object_item_single_inline.h`](object_item_single_inline.h.md)
**Tier floor** — T2.

## Purpose

Declares the surface implemented in
[`object_item_single_inline.h`](object_item_single_inline.h.md): one entry shape, selected
by which base the registered class descends from.

## Exported units

- **single entry** — holds one class and knows which half it is. Asked for the half it does
  not have, it aborts.
