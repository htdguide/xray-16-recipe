# src/xrGame/WeaponUpgrade.cpp

> Applies one purchased weapon upgrade by re-reading a configuration section over the weapon's already-loaded parameters, either for real or as a dry run that only reports which parameters the upgrade would touch.

**Needs** — [`Weapon.h`](Weapon.h.md) · [`inventory_item.h`](inventory_item.h.md) · [`inventory_item_inline.h`](inventory_item_inline.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: configuration reads and arithmetic on already-loaded fields; nothing device- or layout-facing

## Purpose

An upgrade in the shipped games is not a new weapon: it is a small configuration section
naming a handful of the parent weapon's keys, applied on top of the weapon that is
already in the player's hands. This file is the weapon-specific half of that mechanism —
the generic item half lives in [`inventory_item.h`](inventory_item.h.md) — and it exists
as a separate file purely to keep the several hundred lines of parameter plumbing out of
the weapon's main implementation. The split is arbitrary and a rebuild may merge it.

The interesting decisions here are not the key names but the three rules that govern how
a key is applied: **add or replace**, **degrees or raw**, and **structural or scalar**.

## State

`Stateless.` Every routine mutates the weapon it is called on.

## `install_upgrade_impl`

**Contract** — takes an upgrade section name and a *test* flag; returns whether the
section named anything this weapon understands. With the test flag set nothing is
written: the call only answers "does this upgrade affect me", which is what the upgrade
screen uses to grey out an entry and what validation uses to catch a typo in an upgrade
section. Runs the generic item pass first, then four weapon passes, and the result is the
disjunction — *any* recognised key counts.

**Invariants** — a test run and a real run must recognise exactly the same key set; the
two paths differ only in whether they assign. This is why every parameter goes through
the same two helpers rather than being read directly.

```text
FUNCTION install_upgrade(section, test) -> bool
  touched  = item_level_upgrade(section, test)   # condition, cost, weight, …
  touched |= upgrade_ammo_class(section, test)
  touched |= upgrade_dispersion(section, test)
  touched |= upgrade_hit(section, test)
  touched |= upgrade_addon(section, test)
  RETURN touched
```

**Notes** — the four passes are evaluated unconditionally, never short-circuited, because
each has a side effect and the caller wants all of them.

## The two application rules

Two helpers, inherited from the item layer, cover almost every key:

```text
FUNCTION apply_additive(section, key, target, test) -> bool
  IF key absent from section OR its text is empty THEN RETURN false
  IF NOT test THEN target = target + parsed(section, key)
  RETURN true

FUNCTION apply_replacing(section, key, target, test) -> bool
  IF key absent from section OR its text is empty THEN RETURN false
  IF NOT test THEN target = parsed(section, key)
  RETURN true
```

The choice between them is the load-bearing part of every line in this file. A *quantity*
(dispersion, recoil angle, hit power, fire distance) is **additive**, so that upgrades
compose: buying two accuracy upgrades applies both deltas, and an upgrade section carries
a signed delta, not a final value. A *mode* or *identity* (ammunition list, scope status,
silencer name, zoom-enabled flag) is **replacing**, because adding two of those is
meaningless.

A present-but-empty value is treated as absent, which lets an upgrade section inherit
from a parent and blank out a key it does not want.

Angles are a third case: every camera-recoil and dispersion angle is authored in degrees
and stored in radians, so its additive helper converts before adding. A rebuild that
stores angles in one unit throughout still owes the conversion at the parse boundary,
because the shipped upgrade sections are in degrees.

## `install_upgrade_ammo_class`

**Contract** — replaces the weapon's magazine capacity (additive) and its list of
acceptable ammunition sections (replacing). The ammunition value is a comma-separated
tuple of section names; on a real run the existing list is discarded, rebuilt from the
tuple, and the selected ammunition index is reset to the first entry — necessary because
the previously selected index may not exist in the new list.

## `install_upgrade_disp`

**Contract** — the accuracy and handling block: base dispersion, the condition-wear
dispersion factor, effective fire distance, the two camera-recoil parameter sets (hip and
zoomed), the movement-dependent dispersion model, the misfire model, per-shot condition
wear, and the zoom-enabled flag.

**Invariants** — after the pass, relaxation speed and the two maximum recoil angles must
be non-zero in both recoil sets. A zero relaxation speed means the camera never returns
from recoil and a zero maximum angle divides by zero in the recoil accumulator, so these
are checked rather than clamped: a configuration that produces them is a data bug.

**Notes**

- The two boolean camera flags (whether the camera returns after recoil, whether the
  return stops when the player moves the view) are round-tripped through a small integer
  because the replacing helper is written against the configuration reader's typed
  accessors and there is no boolean accessor in that family — incidental, and a rebuild
  reads them as booleans directly.
- The hip and zoomed recoil sets are the same record read from two key prefixes. A
  rebuild should express that as one parameter block read twice, not as duplicated code.

## `install_upgrade_hit`

**Contract** — damage, impulse, muzzle velocity, the aimed-first-shot option and rate of
fire.

**Invariants** — hit power is authored per difficulty level as a tuple of up to four
values, ordered master, veteran, stalker, novice. A shorter tuple is *not* an error: every
missing level takes the master value, so a single-valued tuple means "the same damage at
every difficulty". Rate of fire is authored in rounds per minute but stored as
seconds-per-shot, so the pass converts and must reject a zero.

**Notes**

- Damage and critical damage are *replacing*, not additive, even though they are
  quantities — an upgrade restates the whole difficulty tuple. This is the one place the
  add/replace rule above is broken, and it is broken because a tuple has no meaningful
  sum.
- **Defect, recoverable from the source**: the critical-damage pass reads the
  `hit_power_critical` key into the same variable the ordinary hit-power pass used, then
  parses the *critical* variable, which is still empty. The effect is that every
  difficulty's critical damage becomes zero whenever an upgrade names that key. A rebuild
  should parse the key it read.
- The rate-of-fire default is derived from the weapon's current seconds-per-shot before
  the pass, so an upgrade that omits the key leaves the timing untouched.

## `install_upgrade_addon`

**Contract** — the attachment block: scope, silencer and grenade launcher. Each has a
three-valued status — absent, permanently built in, or attachable — and the status is
read first because it decides whether the rest of that attachment's keys are read at all.

```text
FOR EACH attachment IN (scope, silencer, grenade_launcher)
  status_changed = apply_replacing(section, "<attachment>_status", status, test)
  IF status_changed AND NOT test AND status IS attachable OR permanent THEN
    read the attachment's own parameters
    IF status IS permanent THEN
      attach it immediately        # a built-in attachment is never chosen by the player
```

For the scope specifically, the attachable case reads a *list* of scope sections the
weapon will accept; when that list is missing the upgrade section itself is taken as the
single acceptable scope. The permanent case appends the upgrade section as the scope and
re-runs attachment initialisation on the spot, so that the weapon's model, zoom factor and
crosshair change the moment the upgrade is installed rather than at the next respawn.

**Notes**

- The scope-dependent range and field-of-view modifiers, the dynamic-zoom flag, the
  night-vision post-process name and the alive-detector (binocular) name are all part of
  the scope block; they are read outside the status guard so that an upgrade may adjust a
  scope the weapon already has.
- Silencer and grenade-launcher parameters (model name and inventory-grid coordinates) are
  read with the plain configuration reader rather than the guarded helper: naming the
  status without the accompanying keys is a data error and faults there rather than
  half-applying.
