# src/xrGame/BoneProtections.h

> Declares the per-bone armour table built in [`BoneProtections.cpp`](BoneProtections.cpp.md).

**Needs** — [`BoneProtections.cpp`](BoneProtections.cpp.md)
**Used by** — [`ActorHelmet.cpp`](ActorHelmet.cpp.md) · [`ActorHelmet.h`](ActorHelmet.h.md) · [`BoneProtections.cpp`](BoneProtections.cpp.md) · [`CustomOutfit.cpp`](CustomOutfit.cpp.md) · [`CustomOutfit.h`](CustomOutfit.h.md) · [`ai_stalker_fire.cpp`](ai/stalker/ai_stalker_fire.cpp.md)
**Tier floor** — T3: a declaration and one enumeration

## Purpose

Declares the record every piece of armour in the game carries. Substance is in
[`BoneProtections.cpp`](BoneProtections.cpp.md); the load-bearing content *here* is the
**fraction-kind enumeration**, which is the engine's record of which of the three shipped
games a given armour's numbers were balanced for.

Exported units:

- `SBoneProtections` — the table: a map from bone index to (coefficient, armour,
  pass-through flag), plus the minimum damage fraction and its kind.
- `SBoneProtections::BoneProtection` — one bone's entry. Coefficient defaults to one
  (no reduction), armour to zero (stops nothing), pass-through to false.
- The two configuration key names the parser must skip, `hit_fraction` and
  `hit_fraction_npc` — named here because they are frozen data keys, not identifiers.
- `reload` / `add` — build the table, or merge a second section onto it additively.
- `getBoneProtection` / `getBoneArmor` / `getBonePassBullet` — the three total lookups.

## The four fraction kinds

Each names a damage formula, not merely a source of a number:

- **oldest** — the original game's, keyed by the generic configuration key. Protection
  values are stored as pass-through fractions and must be inverted on read.
- **newer actor** — introduced in the second game, assigned by the outfit or helmet rather
  than read from the armour section, and read from a key on the *item's* section instead.
- **creature** — introduced in the third game, keyed by the creature-specific configuration
  key, so that a creature's own armour is distinguishable from a separate per-creature
  damage factor the same data files carry.
- **newest actor** — the third game's actor formula, again assigned externally.

The default is the creature kind, because the majority of users of this table are
creatures and they are the ones with no external assignment.

## Notes

**The kind can be set by the owner before the table is built, and if it is, the builder
must not overwrite it.** That handshake is how an outfit forces its own generation's
formula onto a shared parser. A rebuild that passes the generation in as an argument
expresses the same thing more clearly.
