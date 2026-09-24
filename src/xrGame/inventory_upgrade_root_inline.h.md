# src/xrGame/inventory_upgrade_root_inline.h

> The one accessor of an upgrade root: the name of the scheme that lays its upgrades out on screen.

**Needs** — [`inventory_upgrade_root.h`](inventory_upgrade_root.h.md)
**Used by** — [`inventory_upgrade_root.h`](inventory_upgrade_root.h.md)
**Tier floor** — T3: a field read

## Purpose

Carries `Root::scheme` out of the declaration, following the namespace's file convention.
A rebuild folds it into the root node's declaration.

## State

`Stateless.`

## `scheme`

**Contract** — returns the name of the layout scheme authored for this item: the data that
tells the upgrade screen how many cells the grid has and where each one sits. Upgrades
address cells in it by their scheme index.
