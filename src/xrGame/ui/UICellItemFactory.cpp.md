# src/xrGame/ui/UICellItemFactory.cpp

> The single place that decides which kind of cell widget an inventory item gets.

**Needs** — [`UICellCustomItems.h`](UICellCustomItems.h.md) · [`UICellItem.h`](UICellItem.h.md)
**Used by** — [`UICellItemFactory.h`](UICellItemFactory.h.md)
**Tier floor** — T3: a three-way dispatch on item kind

## Purpose

Every screen that shows inventory — the inventory panel, the trade panel, the loot panel, the
quick slots — builds its cells through this one function, so the mapping from item kind to
cell behaviour is decided once and cannot drift between screens. That is the whole reason it
is a separate file: a rebuild that inlines the dispatch at each call site will eventually have
a screen where ammunition does not stack.

## State

`Stateless.`

## `create_cell_item`

**Contract** — Given an inventory item, produce the cell widget that represents it. The choice
is by item kind, most specific first: ammunition gets the cell that sums box counts across a
stack, a weapon gets the cell that draws attachment icons over the weapon icon, and anything
else gets the plain inventory cell. The returned widget holds the item as its payload and is
owned by the caller.

```text
FUNCTION create_cell_item(item) -> CellWidget
  IF item is ammunition THEN RETURN AmmoCell(item)
  IF item is a weapon   THEN RETURN WeaponCell(item)
  RETURN InventoryCell(item)
```

**Notes** — The order matters and is not arbitrary: ammunition is tested before weapon because
the two kinds are disjoint in this codebase but the test is a downcast, and putting the rarer,
cheaper test first is the convention the file follows. More load-bearing is that there is no
default registration table here: the set of cell kinds is closed and known at build time,
unlike the entity class factory. A rebuild is free to make it a table, but nothing in the
shipped data names a cell kind, so nothing forces it to.
