# src/xrGame/inventory_upgrade_manager_inline.h

> Empty.

**Needs** — _none_
**Used by** — [`inventory_upgrade_manager.h`](inventory_upgrade_manager.h.md)
**Tier floor** — T4: nothing to compile

## Purpose

The file exists to complete the naming convention every class in this namespace follows —
declaration, inline accessors, implementation — but the upgrade manager kept no accessors
small enough to be worth inlining, so the body is an empty namespace. A rebuild does not
create this file.

## State

`Stateless.`
