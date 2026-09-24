# src/xrGame/medkit.cpp

> An edible item under its own class identifier, with no behaviour of its own.

**Needs** — [`medkit.h`](medkit.h.md) · [`eatable_item_object.h`](eatable_item_object.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a class identifier slot

## Purpose

Healing is not implemented here. An *edible item* applies a set of condition changes when
consumed — health, radiation, bleeding, satiety — and every one of those numbers comes from
the item's configuration section. A medical kit is therefore fully described by its data, and
this class adds nothing to the base behaviour at all.

It exists because the spawn format selects a class by a fixed tag, and because scripts
type-test against it: `is this item a medical kit` is asked, and is only answerable if the
type is distinct. The empty constructor and destructor are the C++ minimum for a distinct
type; a rebuild whose factory can map a tag to a base class with a section needs no type here
at all — unless its script layer also wants the type test, which the shipped scripts do.

## State

`Stateless.` — everything the item has belongs to the edible-item base and its configuration
section.

## `CMedkit`

**Contract** — construction and destruction, both empty. The class's entire contribution is
its identity.
