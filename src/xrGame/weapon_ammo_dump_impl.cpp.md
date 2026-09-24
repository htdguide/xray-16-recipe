# src/xrGame/weapon_ammo_dump_impl.cpp

> Writes one chambered round's effective ballistics back out as a configuration section, for the balance-tuning tools.

**Needs** — [`WeaponAmmo.h`](WeaponAmmo.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: field-by-field write into the configuration writer

## Purpose

The other half of the balance-tool round trip begun in
[`shootingObject_dump_impl.cpp`](shootingObject_dump_impl.cpp.md): the firing object
supplies the weapon's numbers, this supplies the cartridge's. Split into its own file for
the same reason — the tools link the dump without linking the ammunition implementation.

## State

`Stateless.`

## `dump_active_params`

**Contract** — given a section name and a configuration writer, writes the cartridge's
seven live ballistic coefficients into that section. Extends an existing section rather
than replacing it. Reads nothing back.

The keys written, and what each is:

| key | meaning |
|---|---|
| `k_dist` | multiplier on the weapon's maximum range |
| `k_disp` | multiplier on the cone of fire |
| `k_hit` | multiplier on damage |
| `k_impulse` | multiplier on the physical impulse a hit transfers |
| `k_ap` | armour penetration, as a fraction of protection ignored |
| `k_airres` | air resistance, which decides how fast the projectile bleeds speed and drops |
| `k_buckshot` | how many pellets one trigger pull produces — an integer, not a multiplier |

**Invariants** — the names are exactly the names the configuration reader uses, so the
output is loadable as input. That round trip is the file's whole contract.

**Notes** — Six of the seven are *multipliers applied to the weapon's* numbers, not absolute
values. That is the ammunition model in one line: a cartridge does not have a damage, it has
a factor on whatever gun it is fired from. Only the pellet count is absolute, and it is
absolute because it is a count of projectiles rather than a property of one.

The pellet count being written as a signed integer is incidental; it is never negative.
