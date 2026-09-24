# src/xrGame/inventory_owner_inline.h

> One accessor: an inventory owner's trade parameters.

**Needs** — [`InventoryOwner.h`](InventoryOwner.h.md) · [`trade_parameters.h`](trade_parameters.h.md)
**Used by** — [`InventoryOwner.h`](InventoryOwner.h.md)
**Tier floor** — T2: an accessor

## Purpose

Separated from the owner's declaration so the trade parameters' full declaration is not
needed by everything that declares an owner. Incidental; a rebuild may fold it in.

## State

`Stateless.`

## `trade_parameters`

**Contract** — the owner's trade parameters, asserted present.

**Invariants** — every inventory owner has trade parameters, including those who never trade:
the parameters are what *refuse* the trade, so their absence is a construction bug rather
than a state a caller should handle.
