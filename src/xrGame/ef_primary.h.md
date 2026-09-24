# src/xrGame/ef_primary.h

> Declares the thirty primary evaluation functions and, in their constructors, the declared range of each — which is the numbering the trained tables were fitted against.

**Needs** — [`ef_base.h`](ef_base.h.md)
**Used by** — [`ai_stalker_alife.cpp`](ai_stalker_alife.cpp.md) · [`ef_pattern.cpp`](ef_pattern.cpp.md) · [`ef_primary.cpp`](ef_primary.cpp.md) · [`ef_storage.cpp`](ef_storage.cpp.md)
**Tier floor** — T3: a declaration carrying a table of constants

## Purpose

Declares the leaf functions of the evaluation system: the ones that read a property directly
off a creature or an item rather than looking it up in a trained table. Bodies are in
[`ef_primary.cpp`](ef_primary.cpp.md).

The header is not merely a declaration, because each constructor fixes that function's
**name** and **declared range**, and the range is what
[discretization](ef_base.h.md) divides. Those numbers appear nowhere else and are frozen by
the shipped trained data.

## The range table

Each function's minimum and maximum, as declared. A range whose endpoints are small integers
is a *categorical* function: its answer is an enumeration value and the range is the number
of categories, so the bucket count must equal the range for the mapping to be the identity.

| Function | Range | Kind |
|---|---|---|
| `Distance` | 3 – 20 | metres between member and enemy |
| `PersonalHealth` | 0 – 100 | overwritten per evaluation with the creature's own maximum |
| `PersonalMorale` | 0 – 100 | |
| `PersonalCreatureType` | 1 – 21 | categorical: twenty-one creature kinds |
| `PersonalWeaponType` | 1 – 12 | categorical |
| `PersonalAccuracy` | 0 – 100 | |
| `PersonalIntelligence` | 0 – 100 | |
| `PersonalRelation` | 0 – 100 | *unimplemented* |
| `PersonalGreed` | 0 – 100 | *unimplemented* |
| `PersonalAggressiveness` | 0 – 100 | *unimplemented* |
| `PersonalEyeRange` | 0 – 100 | |
| `PersonalMaxHealth` | 0 – 1000 | non-linear bucketing; see the implementation |
| `EnemyMorale` | 0 – 100 | declared, never constructed |
| `EnemyEquipmentCost` | 0 – 12 | *unimplemented* |
| `EnemyRukzakWeight` | 1 – 12 | carried weight |
| `EnemyAnomality` | 1 – 12 | *unimplemented* |
| `EnemyAnomalyType` | 0 – 7 | categorical |
| `EnemyDistanceToGraphPoint` | 0 – 4 | already bucketed by the implementation |
| `GraphPointType0` | 0 – 100 | terrain type of the enemy item's graph vertex |
| `EquipmentType` | 1 – 5 | categorical |
| `ItemDeterioration` | 0 – 100 | |
| `EquipmentPreference` | 1 – 3 | |
| `MainWeaponType` | 1 – 4 | categorical |
| `MainWeaponPreference` | 1 – 3 | |
| `ItemValue` | 100 – 2000 | currency |
| `WeaponAmmoCount` | 0 – 10 | non-linear bucketing; see the implementation |
| `DetectorType` | 0 – 2 | categorical |

**Invariant** — the categorical ranges are *counts*, so a rebuild adding a creature kind must
widen the corresponding range and refit every trained table that reads it. They are not
extensible in practice.

## The two hand-written discretizations

Two functions override the linear bucketing declared in [`ef_base.h`](ef_base.h.md), and one
does so here in the header:

- **`Distance`** collapses the interior to a **single** bucket: below the minimum is bucket
  zero, above the maximum is the top bucket, and everything between is bucket one regardless
  of the requested resolution. So distance is effectively *near / mid / far* whatever the
  trained table asked for. A rebuild that implements the linear rule here changes the
  behaviour of every trained function that takes distance as an input.
- **`PersonalMaxHealth`** and **`WeaponAmmoCount`** override in
  [`ef_primary.cpp`](ef_primary.cpp.md), with explicit break points.

## Exported units

One class per function in the table, each supplying a constructor that sets its name and
range, and an implementation of the value query. Three also expose helpers:
`PersonalWeaponType` exposes the weapon-type read and the best-weapon search;
`PersonalMaxHealth` and `WeaponAmmoCount` expose their discretization overrides.
