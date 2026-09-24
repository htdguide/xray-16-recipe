# src/xrGame/script_binder_object.h

> Declares the script-side binder base class.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_binder.cpp`](script_binder.cpp.md) · [`script_binder.h`](script_binder.h.md) · [`script_binder_inline.h`](script_binder_inline.h.md) · [`script_binder_object.cpp`](script_binder_object.cpp.md) · [`script_binder_object_script.cpp`](script_binder_object_script.cpp.md) · [`script_binder_object_wrapper.cpp`](script_binder_object_wrapper.cpp.md) · [`script_binder_object_wrapper.h`](script_binder_object_wrapper.h.md) · [`script_game_object_script2.cpp`](script_game_object_script2.cpp.md)
**Tier floor** — T2: a declaration of a dispatch surface

## Purpose

Declares the surface implemented in [`script_binder_object.cpp`](script_binder_object.cpp.md):
one public field and the eleven overridable lifecycle hooks. Kept separate so the many
call sites that merely *hold* a binder need not see how the hooks dispatch.

Exported units:

- `object` — the bound game object facade, readable and writable from script.
- `construct(object)` / destructor.
- `reinit`, `reload`, `net_spawn`, `net_destroy`, `net_import`, `net_export`,
  `shedule_update`, `save`, `load`, `net_save_relevant`, `net_relcase` — the hooks; see the
  implementation twin for their contracts and their ordering.
- A registration entry point that exports the class to the script layer.
