# src/xrGame/inventory_item_impl.h

> One accessor, separated so that an item can reach its owner without every item header depending on the inventory.

**Needs** — [`Inventory.h`](Inventory.h.md) · [`inventory_item.h`](inventory_item.h.md)
**Used by** — [`inventory_item.cpp`](inventory_item.cpp.md) · [`inventory_item_upgrade.cpp`](inventory_item_upgrade.cpp.md)
**Tier floor** — T2: an accessor

## Purpose

An item needs its *owner* — the creature or player carrying it — to ask permission questions
(may I be traded, who is my reputation attached to). Reaching the owner means going through
the inventory, and depending on the inventory's full declaration from every item header would
be circular. So the one accessor that needs it lives here, included only by implementation
files.

The split is purely a dependency artifact; a rebuild has no need of the file.

## State

`Stateless.`

## `inventory_owner`

**Contract** — the owner of the inventory this item is in. Asserts both that the item is in an
inventory and that the inventory has an owner.

**Invariants** — an item with no inventory has no owner, and asking is a programming error
rather than a condition. Callers test for an inventory first; see the trade-permission query
in [`inventory_item.cpp`](inventory_item.cpp.md), whose source carries a standing question
about why that test is needed at all.
