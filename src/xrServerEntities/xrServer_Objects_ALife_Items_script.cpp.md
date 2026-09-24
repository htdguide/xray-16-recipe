# src/xrServerEntities/xrServer_Objects_ALife_Items_script.cpp

> Exports the inventory-item mixin, the item record and the weapon family to scripts, including the addon vocabulary and the magazine accessors.

**Needs** — [`xrServer_Objects_ALife_Items.h`](xrServer_Objects_ALife_Items.h.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

The item half of the record export. Most of it is a list of registrations with no added
surface — the point being that a script class may subclass any item record and override its
serialization. Two registrations carry real surface, and both concern things a mod script
genuinely manipulates: upgrades and ammunition.

## `cse_alife_inventory_item`

**Contract** — the mixin, with two methods: does this item have a named upgrade installed,
and install one.

**Invariants** — installing an upgrade that is already present is a **hard failure** on the
native side (see
[`xrServer_Objects_ALife_Items.cpp`](xrServer_Objects_ALife_Items.cpp.md)), and the export
does nothing to soften it. A script that installs blindly must test first — which is
precisely why the test is exported alongside.

**Notes** — both are exported through small adapters rather than directly, because the native
signatures take an interned string and the script side passes plain text. No behavioural
difference.

The constructor is present and commented out: a mixin cannot be built on its own.

## `cse_alife_item_weapon`

**Contract** — the weapon record, plus:

- **the addon vocabulary**, as a script enumeration in **two halves under one name**: the
  three addon *kinds* (grenade launcher, scope, silencer) and the three addon *states*
  (attachable, disabled, permanent). Together they describe what a weapon section permits
  and what this weapon currently has.
- **`clone_addons`** — copy another weapon's addon configuration onto this one, which is how
  a script duplicates a weapon.
- **`set_ammo_elapsed` / `get_ammo_elapsed`** — the rounds currently in the magazine.
- **`get_ammo_magsize`** — the magazine's capacity, from the section.

**Invariants** — **the two enumerations share one script namespace**, so a kind value and a
state value can be compared to each other without complaint. They are different vocabularies
and the collision is a naming accident frozen by conformance criterion 10.

The rounds count is writable and the capacity is not: capacity is the section's, count is the
record's. That split is the chapter's rule about what a record stores, expressed in two
accessors.

## The remaining registrations

`cse_alife_item` (a dynamic visual object that is an inventory item — the base of every item
below), `cse_alife_item_torch`, `cse_alife_item_ammo`, `cse_alife_item_weapon_shotgun`,
`cse_alife_item_weapon_auto_shotgun`, `cse_alife_item_detector`, `cse_alife_item_artefact`.

All at the **item** level, which adds "is this still worth keeping" to the dynamic-alife set.

**Notes** — the item record's registration carries a commented-out alternative registering it
at the **abstract** level instead. Choosing the item level is what gives every item record
the keep-or-discard override, and a mod's item that never answers it is kept forever.
