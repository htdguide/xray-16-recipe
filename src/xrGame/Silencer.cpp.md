# src/xrGame/Silencer.cpp

> The silencer attachment: an inventory item with an explicit, fully empty lifecycle.

**Needs** — [`Silencer.h`](Silencer.h.md) · [`inventory_item_object.h`](inventory_item_object.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a class identifier with a name

## Purpose

A silencer's whole effect — the reduced report, the changed muzzle flash, the hit-power
and dispersion modifiers, the condition it wears at — is configuration the *weapon* reads
when the attachment is fitted. The attachment object itself only has to exist, be
carryable and be tradeable, all of which the generic inventory item already does.

What is interesting about the file is that it overrides seven lifecycle methods and each
one does nothing but call the version it overrides. That is not behaviour; it is a
**checklist** — the author writing out the full set of hooks an inventory item has, as a
place to put the silencer-specific handling that was never needed. A rebuild should keep
the list as documentation of the lifecycle and write none of the overrides.

The lifecycle it enumerates, in the order the hooks fire, is the recurring shape of this
whole directory: load the configuration section, spawn from the server record, become a
child of an inventory owner, update each frame while held, become independent again when
dropped, be destroyed.

## State

`Stateless.`

## `CSilencer`

**Contract** — a plain inventory item. Every override — `Load`, `net_Spawn`, `net_Destroy`,
`UpdateCL`, `OnH_A_Chield` (the object has just become a child of an owner),
`OnH_B_Independent` (the object is about to stop being a child, with a flag saying whether
this is happening only because it is being destroyed) — delegates unchanged.

**Notes** — the distinction between the two attachment hooks is worth keeping even though
this class does not use it: one fires *after* attachment is complete, the other *before*
detachment, so that an item can still see its owner on the way out. Getting that ordering
wrong is a common source of dangling owners.
