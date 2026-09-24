# src/xrGame/hit_immunity.cpp

> The per-damage-type multiplier table: how an outfit, a creature or a vehicle resists each kind of damage differently.

**Needs** — [`hit_immunity.h`](hit_immunity.h.md) · [`hit_immunity_space.h`](hit_immunity_space.h.md) · [`alife_space.h`](../xrServerEntities/alife_space.h.md)
**Used by** — reached through its declarations in [`hit_immunity.h`](hit_immunity.h.md); callers name that, not this file.
**Tier floor** — T2: a table read from configuration and multiplied into a damage value

## Purpose

Damage in this game is typed — a bullet, a burn, a shock, a dose of radiation and a telepathic
attack are different things — and almost every object that can be hurt resists the types
differently. This file is the whole mechanism: a table of multipliers indexed by damage type,
filled from a configuration section, applied by multiplication.

The interesting decisions are all in how the table is *filled*: which configuration keys map
to which damage types, what happens when a key is missing, and the two different fill modes
that let a helmet's protection stack on top of an outfit's.

## State

```text
RECORD HitImmunity
  multipliers : list<real>   # one per damage type; ALL slots start at 1.0
```

**Invariants** — the table is full-length and every slot is meaningful from construction. A
damage type nobody configured multiplies by one, so an object with no immunity section takes
every kind of damage unmodified. There is no "unset" state and no branch on one at the point
of use, which is what keeps the hit path a single multiply.

## The configuration key map

The frozen correspondence between damage type and configuration key. It ships in the game
data, so a rebuild may not rename a key:

```text
burn            <- "burn_immunity"
strike          <- "strike_immunity"
shock           <- "shock_immunity"
wound           <- "wound_immunity"                    # melee / blunt trauma
radiation       <- "radiation_immunity"
telepathic      <- "telepatic_immunity"                # spelling is frozen
chemical_burn   <- "chemical_burn_immunity"
explosion       <- "explosion_immunity"
fire_wound      <- "fire_wound_immunity"               # bullets
light_burn      <- "burn_immunity"                     # shares burn's key
physic_strike   <- "physic_strike_wound_immunity"      # optional
```

**Invariants** — two entries are load-bearing anomalies:

- **Light burn shares the burn key.** There is no separate configuration for it anywhere in
  the shipped data; a light burn is resisted exactly as a burn is. A rebuild that gives it
  its own key will read nothing and leave the multiplier at one, which makes anomaly damage
  noticeably harsher.
- **Physical strike is the only optional key.** Every other key must be present or loading
  fails loudly. The optionality exists because physical strike — damage from being hit by a
  moving rigid body — was added after the data was authored, so most shipped sections lack
  it and must keep their multiplier of one.

The misspelling of the telepathic key is in the shipped data and is therefore frozen.

## `LoadImmunities`

**Contract** — replaces the table from a named configuration section. The section must exist;
a missing section is fatal, because an object that declares an immunity section and does not
have one is a data error. Every non-optional key must be present.

```text
FUNCTION load_immunities(section, config, invert)
  REQUIRE config has section
  FOR EACH (damage_type, key, optional) IN key_map
    IF optional AND config has no `key` in `section` THEN CONTINUE
    value = config.read_real(section, key)
    IF invert THEN value = 1.0 - value
    multipliers[damage_type] = value          # REPLACE
```

**Invariants** — replacement, not accumulation: calling this twice leaves only the second
section's values.

**Notes** — the inversion flag exists because the data expresses the same idea two ways. Some
sections give a *multiplier* ("this outfit passes 0.3 of a burn") and others give a
*protection fraction* ("this outfit stops 0.7 of a burn"). Rather than normalize the data,
the caller says which convention its section uses. A rebuild may normalize instead, but must
then convert the shipped data.

Inversion is unclamped, so a protection value above one produces a negative multiplier and a
hit that *heals*. Nothing in the shipped data does this, and nothing guards against it.

## `AddImmunities`

**Contract** — accumulates a section into the table by addition. Every key is treated as
optional here, regardless of the map's optionality flag: a section being layered on top is
expected to mention only what it changes.

```text
FUNCTION add_immunities(section, config, invert)
  REQUIRE config has section
  FOR EACH (damage_type, key) IN key_map
    IF config has no `key` in `section` THEN CONTINUE
    value = config.read_real(section, key)
    IF invert THEN value = 1.0 - value
    multipliers[damage_type] += value         # ACCUMULATE
```

**Invariants** — this is how a helmet's protection combines with an outfit's, and it is
**additive on the multiplier**, not multiplicative. Two sources each passing 0.5 of a hit
combine to pass 1.0 of it, not 0.25 — layering armour this way makes an object *less*
protected. A rebuild must reproduce the addition to match shipped balance, but should
understand that the base table already starts at one, so any accumulation is on top of a
full-damage baseline and the caller is expected to have loaded a replacing section first.

**Notes** — the light-burn entry shares the burn key here too, so a section that names
`burn_immunity` adds to *two* slots.

## `GetHitImmunity` and `AffectHit`

**Contract** — the multiplier for a damage type, and a damage magnitude multiplied by it.
Both are unconditional table reads with no bounds test: the damage type is an enumeration and
the table is sized to it.
