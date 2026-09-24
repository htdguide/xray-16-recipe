# src/xrGame/CustomOutfit_script.cpp

> Exports the body outfit and the helmet to the script virtual machine, with their protection and condition-restore parameters as directly writable fields.

**Needs** — [`CustomOutfit.h`](CustomOutfit.h.md) · [`ActorHelmet.h`](ActorHelmet.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Two registrations, one for the torso outfit and one for the head outfit, both as
subclasses of the game object facade with a no-argument constructor. Their exported
surfaces are deliberately near-identical, because a script that adjusts a suit's
regeneration rates should not have to branch on which slot it is in.

The notable decision is that the condition-restore rates and the stamina penalty are
exported as *writable* fields, not as getters. Script code is expected to change a worn
suit's behaviour at run time — an upgrade applied at a workbench, a temporary effect —
without going back through configuration. That makes those field names part of the frozen
script surface ([conformance criterion 10](../../SYSTEM-REQUIREMENTS.md#6-conformance)).

## State

`Stateless.`

## `CCustomOutfit::script_register`

**Contract** — registers `CCustomOutfit` deriving from the game object facade,
default-constructible, exposing:

- writable: the stamina penalty, two additive carry-weight bonuses, and the five
  per-second condition-restore rates (health, radiation, satiety, power, bleeding);
- read-only: whether this outfit permits a helmet to be worn alongside it;
- `BonePassBullet` — whether a named body part is unarmoured, so a shot through it
  bypasses the outfit entirely;
- `get_artefact_count` — how many artefact containers the outfit provides;
- `GetDefHitTypeProtection` — the outfit-wide protection against one damage type;
- `GetHitTypeProtection` — protection against one damage type at one skeleton element;
- `GetBoneArmor` — the armour value of one bone.

## `CHelmet::script_register`

**Contract** — registers `CHelmet` with the same shape: the same five restore rates and
stamina penalty as writable fields, and the same three protection queries. It exports
neither the carry-weight bonuses nor the artefact containers, because a helmet has
neither.

## Notes

The two protection queries are exported through small adapters rather than directly. Both
adapt the *same* two mismatches, and both are worth stating because a rebuild hits them
too:

- the damage type is a named enumeration internally but arrives from script as a plain
  number, so the adapter widens then narrows it. A rebuild should export the enumeration
  and delete the adapter.
- the skeleton element arrives from script in a parameter typed as text but carrying a
  bone index. This is a genuine defect preserved for compatibility: shipped scripts pass
  a number, the binding layer's overload resolution routes it through the text
  conversion, and the adapter reinterprets the value back into an index. A rebuild should
  export an integer-typed overload and accept that scripts written against the original
  may pass something strange.
