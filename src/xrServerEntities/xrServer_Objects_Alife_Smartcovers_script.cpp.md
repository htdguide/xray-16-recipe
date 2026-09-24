# src/xrServerEntities/xrServer_Objects_Alife_Smartcovers_script.cpp

> Exports the smart-cover record, with the description name readable and the per-instance loophole table writable.

**Needs** — [`xrServer_Objects_Alife_Smartcovers.h`](xrServer_Objects_Alife_Smartcovers.h.md) · [`xrServer_script_macroses.h`](xrServer_script_macroses.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2.

## Purpose

Registers the smart-cover record at the dynamic-alife level with three methods.

- **`description`** — read the name of the script table describing this cover's interior. A
  script uses it to look the table up itself, which is how the AI decides what the cover
  affords.
- **`set_available_loopholes`** — hand the record a script table saying which of the
  description's loopholes this particular placement offers. This is the per-instance override
  that lets one authored description serve several covers that differ in which positions are
  usable.
- **`set_loopholes_table_checker`** — editor only: bind a property row to the loophole table
  so an author toggling a loophole triggers a reparse.

## Notes

**The loophole table is set, never read back.** The record holds the script table and
consults it during the editor's parse; nothing exports a getter. A script that wants to know
the current override must remember what it set.

**The description name is readable but not writable from script.** It is authored in the
spawn file and belongs to the placement; changing it at run time would invalidate everything
the AI has cached about the cover. The editor changes it through the property model instead,
which knows to reparse.

**The third method is absent from the shipping build**, so a script calling it there fails at
the call. Shipped scripts do not; editor scripts do, and they only ever run in an editor
build.
