# src/xrGame/Scope.h

> Declares the telescopic sight attachment, whose only source is the script registration in [`Scope.cpp`](Scope.cpp.md).

**Needs** — [`inventory_item_object.h`](inventory_item_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`Scope.cpp`](Scope.cpp.md) · [`WeaponMagazined.cpp`](WeaponMagazined.cpp.md)
**Tier floor** — T3: a declaration only

## Purpose

Declares `CScope` as a plain inventory item with no added state, and its script
registration entry point implemented in [`Scope.cpp`](Scope.cpp.md). The sight's
magnification, its zoom behaviour and which weapons accept it are all configuration read
by the weapon, not by this class.

The type is marked as having no further subclasses, which is a statement about the design
rather than about the language: an attachment is a leaf.

Exported units:

- `CScope` — the attachment.
- `script_register` — declares this and the other two attachments to Lua.
