# src/xrGame/purchase_list_inline.h

> Reading and writing the deficit table.

**Needs** — [`purchase_list.h`](purchase_list.h.md)
**Used by** — [`purchase_list.h`](purchase_list.h.md)
**Tier floor** — T3: map lookups

## Purpose

Supplies the accessors declared in [`purchase_list.h`](purchase_list.h.md). The one
decision here: **a section with no recorded deficit reads as `1.0`, not as absent**. That
default is what lets the price path multiply unconditionally, and it means an item that was
never on any shopping list is priced at face value rather than treated as infinitely
scarce.

Writing a deficit replaces an existing entry and inserts otherwise — unlike the restocking
path, which rejects a duplicate, because a script setting a deficit deliberately is
expected to overwrite.
