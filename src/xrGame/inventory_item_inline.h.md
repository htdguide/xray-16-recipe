# src/xrGame/inventory_item_inline.h

> The two configuration-application helpers the upgrade system is built on, plus two accessors.

**Needs** — [`inventory_item.h`](inventory_item.h.md) · [Seam: Configuration format](../xrCore/README.md)
**Used by** — [`WeaponUpgrade.cpp`](WeaponUpgrade.cpp.md) · [`inventory_item.cpp`](inventory_item.cpp.md) · [`inventory_item.h`](inventory_item.h.md)
**Tier floor** — T2: conditional configuration reads

## Purpose

An upgrade is a configuration section that modifies some of an item's tuned values. Applying
one means, for each key the upgrade might name, "if it is present, apply it" — and *before*
applying anything, "if it is present, confirm it is readable". This file is the pair of
operations that make that two-pass structure possible.

## State

`Stateless.`

## `process_if_exists`

**Contract** — if the named key exists in the section and has a non-empty value, either **adds**
its value to the running value or, in test mode, does nothing. Answers whether the key was
present, which is what the caller accumulates.

```text
FUNCTION process_if_exists(section, key, read, value, test_only) -> bool
  IF the key is absent THEN RETURN false
  IF its raw text is empty THEN RETURN false
  IF NOT test_only THEN value = value + read(section, key)
  RETURN true
```

**Invariants** — presence is decided on the **raw text**, not on the parsed value, and an empty
text counts as absent. That is what lets an upgrade section declare a key with no value as a
deliberate no-op, which the shipped data does.

The answer is the same in test mode and in apply mode. That equality is the whole point: the
verification pass and the application pass must agree about which keys exist, or an upgrade
could pass verification and then fail halfway through application, leaving the item partly
upgraded with no way back.

## `process_if_exists_set`

**Contract** — the same, but **replaces** the value instead of adding to it.

**Invariants** — the two exist because upgrades come in two flavours: cumulative (this scope
adds fifty metres of range) and absolute (this barrel sets the calibre). A rebuild must keep
both; which one a given key uses is decided per key at the call site, not in the data, so the
choice is code and cannot be recovered from the configuration.

## `useful_for_NPC`

**Contract** — a creature will pick this item up when it is both generally useful and flagged
useful to creatures. The second flag comes from the spawn record, so a level author can place
an item creatures will ignore.

## `upgardes`

**Contract** — the installed upgrade list, read-only. (The name is misspelled in the original
and is part of the frozen surface.)
