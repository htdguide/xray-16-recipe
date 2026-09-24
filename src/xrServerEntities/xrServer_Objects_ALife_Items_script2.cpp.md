# src/xrServerEntities/xrServer_Objects_ALife_Items_script2.cpp

> Exports the remaining item records: the PDA, documents, throwables, worn protection, and the two magazine-fed weapon levels.

**Needs** — [`xrServer_Objects_ALife_Items.h`](xrServer_Objects_ALife_Items.h.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

The continuation of
[`xrServer_Objects_ALife_Items_script.cpp`](xrServer_Objects_ALife_Items_script.cpp.md),
split for build time. Nine registrations, each at the **item** level, each with **no added
surface**: `cse_alife_item_pda`, `cse_alife_item_document`, `cse_alife_item_grenade`,
`cse_alife_item_explosive`, `cse_alife_item_bolt`, `cse_alife_item_custom_outfit`,
`cse_alife_item_helmet`, `cse_alife_item_weapon_magazined` and
`cse_alife_item_weapon_magazined_w_gl`.

## Notes

**None of these records exposes its own state to script** — not the PDA's owner, not a
document's information piece, not an outfit's protection values, not a magazine-fed weapon's
fire mode. Everything a script needs about an item it reads from the item's configuration
section instead. The registrations exist so a mod can *replace* the record, not read it, and
that asymmetry is the shape of the whole item export.

**The grenade-launcher variant's exported name abbreviates where the type name does not** —
the type spells out the launcher, the script name uses an abbreviation. Frozen by
conformance criterion 10.
