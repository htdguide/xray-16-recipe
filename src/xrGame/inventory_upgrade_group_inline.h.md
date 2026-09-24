# src/xrGame/inventory_upgrade_group_inline.h

> The two field reads of an upgrade group: its identifier, and that identifier as text.

**Needs** — [`inventory_upgrade_group.h`](inventory_upgrade_group.h.md)
**Used by** — [`inventory_upgrade_group.h`](inventory_upgrade_group.h.md)
**Tier floor** — T3: two accessors

## Purpose

A separate file because C++ cannot define a member inside its own class declaration without
committing to inlining policy; the declaration includes this file at its end. No decision
survives the translation.

## `id` / `id_str`

**Contract** — the group's interned configuration section name, and the same as text.
