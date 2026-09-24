# src/xrServerEntities/xrServer_Objects_Alife_Smartcovers.h

> Declares the smart-cover record: an authored place a creature can occupy, whose interior geometry is described in script rather than in the record.

**Needs** — [`xrServer_Objects_ALife.h`](xrServer_Objects_ALife.h.md) · [`xrServer_Objects_Alife_Smartcovers.cpp`](xrServer_Objects_Alife_Smartcovers.cpp.md) · [`script_value_container.h`](script_value_container.h.md)
**Used by** — [`smart_cover_object.cpp`](../xrGame/smart_cover_object.cpp.md) · [`object_factory_register.cpp`](object_factory_register.cpp.md) · [`xrServer_Objects_Alife_Smartcovers.cpp`](xrServer_Objects_Alife_Smartcovers.cpp.md) · [`xrServer_Objects_Alife_Smartcovers_script.cpp`](xrServer_Objects_Alife_Smartcovers_script.cpp.md)
**Tier floor** — T2.

## Purpose

Declares one record. The contracts are in
[`xrServer_Objects_Alife_Smartcovers.cpp`](xrServer_Objects_Alife_Smartcovers.cpp.md).

The record is unusual enough in the chapter to be worth naming here: **almost none of what a
smart cover is lives in the record**. The record holds a volume, a description *name*, and
four tunings. The cover's actual structure — its loopholes, their fields of view, their
firing ranges, the transitions between them, which of them a creature may enter through — is
a **table in the game's script data**, looked up by that name. The record is a placement and
a reference.

That makes it the clearest case in the chapter of data living outside the engine's own
formats, and it is why the editor half of this file is so large: to draw a smart cover, the
editor has to run the script table.

## Exported units

- `CSE_SmartCover` — a dynamic object with a shape. Description name, hold time, the two
  enemy-distance thresholds, the combat-cover and can-fire flags, and a script table of
  enabled loopholes.
- `SSCDrawHelper` — one loophole as the editor needs it: identifier, position, whether it can
  be entered, entry direction, field of view, range, view direction, and an animation name.
  Editor-only and never serialized.

The private half — loophole parsing, enterability analysis, visual construction and
rendering — exists only outside the shipping build.
