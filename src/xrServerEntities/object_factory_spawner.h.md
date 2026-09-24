# src/xrServerEntities/object_factory_spawner.h

> Classifies every configuration section into a browsable category, so a developer tool can offer "spawn one of these" without an authored list.

**Needs** — [`clsid_game.h`](clsid_game.h.md) · [Data: configuration](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`object_factory.h`](object_factory.h.md) · [`object_factory_spawner.cpp`](object_factory_spawner.cpp.md)
**Tier floor** — T3: string hashing and table lookup over configuration values.

## Purpose

The game data has several thousand configuration sections and no index of them. This file
derives one: given a section's `kind`, its `weapon_class` and its class identifier, it
answers which of about thirty categories the thing belongs in — artefact, medicine, pistol,
mutant, anomaly. The classification drives the debug spawner in
[`object_factory_spawner.cpp`](object_factory_spawner.cpp.md) and exists only outside the
shipping build.

Nothing here is required for the game to run. It is included in this chapter because it is
the only place the engine writes down **what a class identifier *means* in game terms** —
which tags are weapons, which are creatures, which are zones. That table is useful to a
rebuilder reading the spawn format even if the tool itself is dropped.

## State

```text
ENUM SpawnCategory
  Artefacts, ArtefactContainers,
  ItemsFood, ItemsDrink, ItemsMedicine, ItemsDevices, ItemsTools, ItemsRepair,
  ItemsParts, ItemsMiscellaneous, ItemsNotes, ItemsQuest, ItemsUpgrades,
  Helmets, OutfitsAttachments, OutfitsLight, OutfitsMedium, OutfitsHeavy,
  WeaponsAmmo, WeaponsMelee, WeaponsPistols, WeaponsShotguns, WeaponsSMG,
  WeaponsRifles, WeaponsSniperRifles, WeaponsExplosives, WeaponsMiscellaneous,
  Vehicles, Physics,
  CreaturesStalkers, CreaturesMutants, CreaturesPhantoms,
  SquadsStalkers, SquadsMutants,
  Zones,
  Unknown
```

**Invariants** — the declaration order is load-bearing twice over: the display list is
iterated in it, and two range tests in
[`object_factory_spawner.cpp`](object_factory_spawner.cpp.md) ask "is this category between
*Artefacts* and *WeaponsMiscellaneous*" (meaning: is it a carryable inventory item) and
"between *WeaponsAmmo* and *WeaponsExplosives*" (meaning: is it a weapon proper). Reordering
the enumeration silently changes both tests. A rebuild should make those two questions
explicit predicates rather than range comparisons.

## `category_label`

**Contract** — the human-readable name of a category, for the tool's list. Pure.

## `category_from_kind`

**Contract** — maps the value of a section's `kind` key to a category, and answers *Unknown*
for anything unlisted. The `kind` key is the game data's own coarse taxonomy
(`i_arty`, `i_medical`, `w_pistol`, `o_heavy` and so on), so this is the most reliable
signal and it is consulted first. Pure.

**Notes** — three mutant kinds from a mod's data (`SM_KARLIK`, `SM_LURKER`, `SM_PSYSUCKER`)
sit in this table beside the shipped ones. They are not in any retail game; they document
that the table is open to extension rather than closed.

## `category_from_class_identifier`

**Contract** — maps a class identifier to a category, covering both the authored tags and
the script shadow tags from
[`object_factory_register.cpp`](object_factory_register.cpp.md). Consulted when `kind` is
absent, which is the case for most of *Shadow of Chernobyl*'s data. Pure.

**Notes** — this table is the closest thing the codebase has to a documented meaning for the
tag set, and it is worth reading alongside [`clsid_game.h`](clsid_game.h.md) for exactly
that. It also reveals a few groupings that are conventions rather than facts: the bolt and
the binoculars are filed under weapon miscellany, and the explosive item is filed under
physics.

## `category_from_weapon_class`

**Contract** — maps a section's `weapon_class` key (`shotgun`, `assault_rifle`,
`heavy_weapon`, `sniper_rifle`) to a weapon category. The fallback when `kind` says nothing
and the class identifier is a generic weapon tag. Heavy weapons are filed with rifles; that
is a tool convenience, not a game fact.

## `detect_category`

**Contract** — the composed rule: try `kind`, then `weapon_class`, then the class
identifier, and answer *Unknown* if all three are silent.

```text
FUNCTION detect_category(kind, weapon_class, identifier) -> SpawnCategory
  c = Unknown
  IF kind EXISTS         THEN c = category_from_kind(kind)
  IF weapon_class EXISTS AND c = Unknown THEN c = category_from_weapon_class(weapon_class)
  IF identifier EXISTS   AND c = Unknown THEN c = category_from_class_identifier(identifier)
  RETURN c
```

**Notes** — the order is the confidence order: the data's own taxonomy beats a derived one,
and a class identifier is the weakest signal because one identifier covers many different
things (every magazine-fed weapon shares a tag).
