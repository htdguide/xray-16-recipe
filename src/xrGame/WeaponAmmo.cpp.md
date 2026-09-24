# src/xrGame/WeaponAmmo.cpp

> A round and a box of rounds: the eleven numbers that decide what a bullet does on impact, and the container that hands them out one at a time.

**Needs** — [`WeaponAmmo.h`](WeaponAmmo.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md) · [`Weapon.h`](Weapon.h.md) · [`Inventory.h`](Inventory.h.md) · [`Level_Bullet_Manager.h`](Level_Bullet_Manager.h.md) · [`xrServer_Objects_ALife_Items.h`](../xrServerEntities/xrServer_Objects_ALife_Items.h.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: the cartridge parameter block is a packed record copied wholesale into every projectile and read back by the anti-cheat dump; its layout is fixed.

## Purpose

Two things live here, and the separation between them is the file's real content.

A **cartridge** is a value, not an object: eleven numbers and a flag set, copied by value
into a magazine and again into each projectile. It has no identity, no position and no
lifetime — which is what lets a magazine be a list of them and a shotgun shell become
eight independent projectiles.

An **ammunition box** is a world object with a count. Its only interesting behaviour is
handing out a cartridge and diminishing.

The cartridge parameters are the *entire* interface between the weapon system and the
bullet system. Everything a bullet does downstream — how far it carries, how hard it
hits, whether it pierces armour, whether it ricochets, how big a hole it leaves — is one
of these numbers multiplied against something the weapon or the material supplies.

## State

```text
RECORD CartridgeParams          # packed: copied into every projectile, dumped for anti-cheat
  distance_factor    : real = 1     # multiplies the weapon's maximum range
  dispersion_factor  : real = 1     # multiplies the weapon's cone
  hit_factor         : real = 1     # multiplies the weapon's damage
  impulse_factor     : real = 1     # multiplies the physical push
  pierce_factor      : real = 1     # older armour model: how much armour it ignores
  armour_piercing    : real = 0     # newer armour model, from the second game onward
  air_resistance     : real         # drag; falls back to a global default
  buckshot           : int = 1      # projectiles per trigger pull of this round
  impair             : real = 1     # multiplies the wear this round costs the weapon
  wallmark_size      : real         # the decal it leaves; must be positive
  tracer_colour_id   : int (8-bit)

RECORD Cartridge
  ammo_section       : text
  params             : CartridgeParams
  local_ammo_type    : int (8-bit)  # index into the weapon's own ammo-type list
  bullet_material    : int (16-bit) # the material identity a bullet presents on impact
  flags              : set of {tracer, ricochet, can_be_unlimited,
                               explosive, magnetic_beam, tracer_one_in_four}

RECORD AmmoBox EXTENDS InventoryItemObject
  params             : CartridgeParams   # the prototype every dispensed round copies
  tracer, tracer_one_in_four : bool
  box_size           : int (16-bit)      # full capacity, from the section
  box_current        : int (16-bit)      # rounds remaining
  ready_to_destroy   : bool
```

**Invariants** —
- `box_current <= box_size`, enforced at spawn by clamping the server record and writing
  the clamp *back* into the record, so a corrupt save is repaired rather than trusted;
- `wallmark_size` is strictly positive — a zero would make an invisible bullet hole;
- `bullet_material` is always the single material named `objects/bullet`, resolved from
  the material library at load;
- a box with zero rounds is not useful and destroys itself when it leaves an inventory.

## `CCartridge::Load` — reading a round

**Contract** — fills a cartridge from a configuration section. Runs whenever a weapon
needs a prototype round of a type, which is at spawn, at reload, and at any ammo-type
switch.

```text
FUNCTION load_cartridge(c, section, local_type_index)
  c.ammo_section = section; c.local_ammo_type = local_type_index
  read distance, dispersion, hit and impulse factors

  # the armour model changed between the first and second games
  IF the material library is at the second game's version or newer THEN
    read armour_piercing        # required
  ELSE
    read pierce_factor          # required
    read armour_piercing, defaulting to 0

  read the tracer colour, defaulting to 0
  air_resistance = the section's own value, or the bullet manager's global default
  read the tracer flag, buckshot count, impair factor and wallmark size

  flags: ricochet and can_be_unlimited default ON, magnetic beam OFF
  IF the section forbids ricochet THEN clear ricochet
  IF the section requests a magnetic beam THEN set it
  read the one-in-four tracer flag if present
  read can_be_unlimited if present
  read explosive, defaulting to false

  bullet_material = the material library's index for "objects/bullet"
```

**Invariants** — the armour-model branch is data compatibility that a rebuild must keep.
The first game's rounds carry a `k_pierce`; the later games' carry a `k_ap`, and the two
are not the same quantity. Which one the bullet system uses is decided by the *material
library's* version, not by the round's, because the armour formula lives in the material
system.

`can_be_unlimited` and `ricochet` default **on**, so a section that says nothing gets
both. The unlimited flag is what stops special ammunition from being free in the
unlimited-ammo modes.

**Notes** — the flags are named for what the bullet system does with them:
*magnetic beam* is the gauss rifle's projectile behaviour, *explosive* the launcher
grenade's, *one-in-four tracer* draws a visible trace for every fourth round rather than
every one.

## `CCartridge::Weight`

**Contract** — one round weighs a box's inventory weight divided by its capacity. There
is no per-round weight in the data; this derivation is the only source of it, and it is
why a partially filled magazine weighs a sensible fraction.

## `CWeaponAmmo::Load`

**Contract** — reads the same parameter block into the box, plus the box capacity, and
starts the box full. Duplicated from the cartridge loader line for line, because the box
keeps a prototype rather than a cartridge. A rebuild should load the block once and hold
one copy.

## `Get` — dispensing a round

**Contract** — copies the box's prototype into the caller's cartridge, stamps the box's
section and tracer flags onto it, decrements the box, and invalidates the owning
inventory's cached state (weight and value both changed). Returns false when the box is
empty, which is the signal the reload loop stops on.

```text
FUNCTION get(box, out cartridge) -> bool
  IF box.box_current = 0 THEN RETURN false
  cartridge.ammo_section    = box's section
  cartridge.params          = box.params
  cartridge.tracer          = box.tracer
  cartridge.one_in_four     = box.tracer_one_in_four
  cartridge.bullet_material = the material library's "objects/bullet" index
  box.box_current = box.box_current - 1
  invalidate the owning inventory's cached state
  RETURN true
```

**Notes** — the dispensed cartridge does **not** carry the box's `local_ammo_type`; the
reload loop stamps that afterward. That is why a cartridge's local type and its section
can disagree for one statement, and why the magazine's type is the weapon's view rather
than the box's.

Also note the ricochet, explosive and magnetic-beam flags are *not* copied: a round
dispensed from a box gets the default flag set (tracer and ricochet) rather than the
section's. Rounds that were authored as non-ricocheting or explosive lose those
properties when they pass through a box — only rounds loaded directly through the
cartridge loader keep them. This is a real behavioural discrepancy and is not
recoverable as intentional from the source.

## `Weight` / `Cost` — partial boxes

**Contract** — both scale linearly with the fraction remaining. Weight scales exactly;
cost scales and rounds to nearest. A box with zero capacity weighs nothing (a guard
against a data error) but its cost divides by zero.

## Lifecycle

**Contract** — spawning adopts the record's elapsed count, clamped to capacity and
written back. Becoming independent (leaving an inventory) destroys an empty box on the
authoritative side and marks it ready for destruction everywhere; a box so marked is not
rendered, so it disappears on the frame it empties rather than on the frame the server
confirms. Per-frame updates interpolate the replicated transform in multiplayer and
assert the transform stays finite — three times, which is the residue of a hunt for a
corruption bug.

The network payload is the base item's plus the current count as a 16-bit value.

## `can_make_killing`

**Contract** — answers "is there a weapon in this inventory that fires me", returning the
first such weapon. It is the mirror of the weapon's `can_kill(inventory)`: the planner
can start from either the gun or the ammunition and find the other.
