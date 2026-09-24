# src/xrServerEntities/xrServer_Objects_ALife_Monsters_script2.cpp

> Exports the crow, the zombie, the generic monster and the stalker.

**Needs** — [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

The second part of the creature export, split for build time. Four registrations, none with
added surface.

- **`cse_alife_creature_crow`** — at the **creature** level, not the monster one: a crow is
  alive but is not scheduled offline and has no brain, so it has no per-tick update to
  override.
- **`cse_alife_monster_zombie`** — at the monster level.
- **`cse_alife_monster_base`** — at the monster level with a ragdoll. **This is the record
  behind every animal in the game**, and a mod that wants a new creature subclasses it.
- **`cse_alife_human_stalker`** — at the monster level, with the human abstract and a ragdoll
  as its bases. The single most subclassed record in the shipped mods.

## Notes

The crow's level is the one decision in this file. Registering it as a creature rather than a
monster is what tells a script subclass that there is no offline update to implement — which
matches the record, where a crow neither switches offline nor occupies a navigation-graph
location.
