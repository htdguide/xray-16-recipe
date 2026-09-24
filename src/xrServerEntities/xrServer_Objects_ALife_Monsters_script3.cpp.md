# src/xrServerEntities/xrServer_Objects_ALife_Monsters_script3.cpp

> Exports the actor, the two remaining zone records, the phantom, the creature level with its health and allegiance, and the squad with its membership surface.

**Needs** — [`xrServer_Objects_ALife_Monsters.h`](xrServer_Objects_ALife_Monsters.h.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

The third part of the creature export. Two registrations carry surface and both matter: the
creature level, which is where a script reads whether something is alive, and the squad,
which is where a script manipulates a group of creatures as one.

## `cse_alife_creature_abstract`

**Contract** — the level that is alive, at the creature level, plus:

- **`health`** — read-only. A script may observe health but not set it; killing goes through
  the monster level's own operation, which unregisters the creature from its squad first.
- **`alive`** — read-only: health above zero. Exported separately so a script never has to
  know the threshold.
- **`team` / `squad` / `group`** — all three **readable and writable**. Writing them
  re-allegiances a creature instantly, which is how a script defects an NPC to another
  faction.
- **`o_torso`** — the torso orientation, handed out **by reference** so a script can modify
  it in place.

**Invariants** — the torso reference is the one place in the record export where a script
holds a pointer into a record's interior. It outlives nothing it should not today, but a
record destroyed while a script holds the reference leaves the script with a dangling one.
A rebuild should hand back a value and take a setter.

**Health is deliberately read-only here and writable nowhere in the export.** The native
setter enforces that health and the recorded killer agree; exposing it would let a script
resurrect something that has a killer, which the simulation treats as an impossible state.

## `cse_alife_online_offline_group`

**Contract** — the squad, at the online/offline-group level, plus a membership surface that
exists only in the game build:

- **`register_member` / `unregister_member`** — add or remove a creature by identity.
- **`commander_id`** — which member leads.
- **`squad_members`** — iterate the membership, as a script iterator over (identity, record)
  pairs.
- **`npc_count`** — how many.
- **`add_location_type` / `clear_location_types`** — set the squad's terrain preference from
  script, one mask at a time.
- **`force_change_position`** — move the whole squad.

Plus a small registered type for one membership entry: its identity and its record, both
read-only.

**Invariants** — the iteration hands out a **live iterator over the squad's own storage**.
Registering or unregistering a member while iterating invalidates it, and nothing detects
that. A script that removes members while walking the squad — which is exactly what a
"disband" routine does — must collect first and remove after.

**The terrain surface is add-and-clear with no read.** A script can replace a squad's
preferences entirely but cannot inspect them. That asymmetry matches how it is used: the
preferences are set once when the squad is created.

**Notes** — a companion operation forcing the squad's game-graph vertex is present and
commented out. Moving a squad's position is exported; moving it between graph vertices is
not, because that would bypass the registries that index entities by location.

The membership entry's script name is spelled with the internal type's name and a double
separator. It is not something a script names deliberately; it exists so the iterator has a
type.

## `cse_alife_creature_actor`

**Contract** — the player record, at the creature level, with its three bases named: a
creature, an identity and a ragdoll. No added surface — everything a script does with the
actor goes through the live object, not the record.

## `cse_torrid_zone` / `cse_zone_visual`

**Contract** — the moving anomaly and the anomaly with a model, both at the dynamic-alife
level with no added surface.

## `cse_alife_creature_phantom`

**Contract** — at the creature level, no added surface.
