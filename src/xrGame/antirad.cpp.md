# src/xrGame/antirad.cpp

> An anti-radiation drug: an edible item with no behaviour beyond what its configuration section gives it.

**Needs** — [`antirad.h`](antirad.h.md) · [`eatable_item_object.h`](eatable_item_object.h.md)
**Used by** — reached through its declarations in [`antirad.h`](antirad.h.md); callers name that, not this file.
**Tier floor** — T3: a named subclass with no added behaviour

## Purpose

A distinct class for the radiation-purging drug. It adds nothing at all to the edible item
it derives from: the actual effect — a negative radiation delta applied on consumption — is
a number in the item's configuration section, read by the base class, and the syringe and
the pills differ only in which section they name.

The class exists because the shipped data's class-identifier table has a tag for it, and
because scripts and the inventory test for an anti-radiation item by class rather than by
section name.

## State

`Stateless.` Everything is inherited.

## `CAntirad`

**Contract** — an edible item under a different name. Construction and destruction are
empty; every path delegates.

**Notes** — a rebuild whose spawn factory can map a class identifier onto "edible item with
this section" needs no class here. Keep the identifier, drop the type.
