# src/xrServerEntities/alife_space.cpp

> Converts hit-type names between the configuration's spelling and the engine's numbering, in both directions.

**Needs** — [`alife_space.h`](alife_space.h.md) · [`xrCore/xr_token.h`](../xrCore/xr_token.h.md)
**Used by** — reached through its declarations in [`alife_space.h`](alife_space.h.md); callers name that, not this file.
**Tier floor** — T3: string comparison against a fixed table.

## Purpose

Damage in this game is typed, and every weapon, anomaly, outfit and script names its types
as strings in configuration (`"fire_wound"`, `"chemical_burn"`). This file is the only
place those strings are bound to the numbering in
[`alife_space.h`](alife_space.h.md). It is a separate translation unit because the tools
build needs the mapping without needing any gameplay code.

## State

```text
RECORD HitTypeName
  name  : text        # exactly as it appears in shipped configuration
  value : HitType
```

The table holds twelve entries: `burn`, `shock`, `strike`, `wound`, `radiation`,
`telepatic`, `fire_wound`, `chemical_burn`, `explosion`, `wound_2`, `physic_strike`,
`light_burn`.

**Invariants** — the spellings are frozen by shipped data, misspelling included:
`telepatic` is the name in every shipped configuration file and must be accepted as written.

## `hit_type_from_name`

**Contract** — case-insensitive lookup of a name. An unrecognized name is **fatal**: it
aborts with "unsupported hit type" rather than falling back to a default. That is the right
severity — a mistyped hit type in a weapon section would otherwise silently make the weapon
deal a different kind of damage, which is invisible until someone notices the armour is not
working.

## `hit_type_to_name`

**Contract** — the inverse, over the same table. Used when writing configuration and in
debug output.

**Notes** — the two directions are implemented twice over: the forward direction is a chain
of comparisons and the reverse reads the table. That duplication is an accident, and a
rebuild should drive both from one table — but note that the chain's order and the table's
order differ, so a rebuild must take the *names* from one of them and not assume the
numbering follows either listing. The numbering is the enumeration's own, in
[`alife_space.h`](alife_space.h.md).
