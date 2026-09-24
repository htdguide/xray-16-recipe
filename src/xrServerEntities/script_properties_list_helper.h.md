# src/xrServerEntities/script_properties_list_helper.h

> Declares the script-facing property factory: the same editor rows, but bound to fields that live in a Lua table rather than in a compiled record.

**Needs** — [`xrEProps.h`](xrEProps.h.md) · [`script_rtoken_list.h`](script_rtoken_list.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`xrgame_dll_detach.cpp`](../xrGame/xrgame_dll_detach.cpp.md) · [`script_properties_list_helper.cpp`](script_properties_list_helper.cpp.md) · [`script_properties_list_helper_script.cpp`](script_properties_list_helper_script.cpp.md)
**Tier floor** — T3.

## Purpose

Declares the surface implemented in
[`script_properties_list_helper.cpp`](script_properties_list_helper.cpp.md).

A script-declared entity class (see [`object_item_script.h`](object_item_script.h.md)) has
its state in a Lua table, and the editor still has to be able to edit it. This is the
adapter: every `create…` call from [`PropertiesListHelper.h`](PropertiesListHelper.h.md)
appears again, taking a script object and a field name instead of a pointer to a field.

## Exported units

The same catalogue as the native helper — caption, canvas, button, choose, the integer
widths, float, boolean, vector, the flag widths, the token widths, list, the colour forms,
text, time, angle and three-component angle — plus the three shared edit hooks, which simply
forward to the native ones with the string types converted.

## Notes

The eight-bit integer family and the interned-token family are declared but commented out.
No script class needed them.
