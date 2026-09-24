# src/xrGame/inventory_upgrade_base_inline.h

> The three field reads of an upgrade node: its identifier, that identifier as text, and whether the player has learned of it.

**Needs** — [`inventory_upgrade_base.h`](inventory_upgrade_base.h.md)
**Used by** — [`inventory_upgrade_base.h`](inventory_upgrade_base.h.md)
**Tier floor** — T3: three accessors

## Purpose

Splitting three one-line accessors into their own file is a C++ habit — a header may not
define a member before the class is complete, so the definitions follow the declaration in a
second file the first includes at its end. It carries no decision. A rebuild has the
accessors wherever the type is declared, or has no accessors at all because the fields are
readable.

## `id` / `id_str` / `is_known`

**Contract** — the node's interned identifier, the same as text, and the learned flag.
