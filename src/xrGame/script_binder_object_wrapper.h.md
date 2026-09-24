# src/xrGame/script_binder_object_wrapper.h

> Declares the script-override adapter for the binder base class.

**Needs** — [`script_binder_object.h`](script_binder_object.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_binder_object_script.cpp`](script_binder_object_script.cpp.md) · [`script_binder_object_wrapper.cpp`](script_binder_object_wrapper.cpp.md)
**Tier floor** — T2: a declaration of a dispatch surface

## Purpose

Declares the surface implemented in
[`script_binder_object_wrapper.cpp`](script_binder_object_wrapper.cpp.md): for each of the
eleven binder hooks, an overriding form that calls into script and a static form that
calls the base implementation. Nothing else. See the implementation twin for the one rule
that generates all twenty-two.
