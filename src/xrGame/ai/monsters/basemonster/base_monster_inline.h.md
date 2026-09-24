# src/xrGame/ai/monsters/basemonster/base_monster_inline.h

> The panic-threshold override pair, separated out so scripts can raise a creature's nerve and put it back.

**Needs** — [`base_monster.h`](base_monster.h.md)
**Used by** — [`base_monster.h`](base_monster.h.md)
**Tier floor** — T3: two assignments

## Purpose

Two operations on one number: set the creature's panic threshold to a value, and restore the
one loaded from its configuration section. The threshold is inherited from the creature
layer above; what this file adds is the *remembered default*, which is what makes the
override reversible.

## `set_custom_panic_threshold`

**Contract** — sets the threshold a script chose. Takes effect on the next state selection.

## `set_default_panic_threshold`

**Contract** — restores the configured value. The default was captured once at load, after
the base class had read the section, which is why it exists as a separate field rather than
being re-read.
