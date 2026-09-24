# src/xrGame/script_hit_script.cpp

> Exports the script hit to the script layer as `hit`, together with the frozen table of damage kinds.

**Needs** — [`script_hit.h`](script_hit.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registration data only

## Purpose

Declares the script-visible shape of a hit. Both the field names and the damage-kind names
are frozen by shipped scripts and by configuration that names kinds in the same spelling.

## `script_register`

**Contract** — registers a class named `hit`, default-constructible and copy-constructible
from another hit, with directly readable and writable fields `power`, `direction`,
`draftsman`, `impulse` and `type`, one method `bone(name)`, and a nested constant table
`hit_type`.

```text
hit.hit_type = {
  burn, shock, strike, wound, radiation, telepatic,
  chemical_burn, explosion, fire_wound, physic_strike, light_burn,
  dummy            # the count, exposed so scripts can bound-check; not a real kind
}
```

**Notes**

The kind names are the *damage channel* names used everywhere else in the data: an
entity's armour is authored as one resistance per kind, and an anomaly declares the kind
it deals. A rebuild must keep the set, the order, and the spellings — `telepatic` included,
misspelling and all, because it appears in shipped configuration.

`dummy` is the one-past-the-end marker exported as if it were a kind. Scripts use it as an
upper bound; nothing may be hit with it.

`bone` is a method while everything else is a field, purely because the name is stored as
interned text rather than a plain value. A rebuild should expose it as a field like the
rest and keep the `bone` spelling.
