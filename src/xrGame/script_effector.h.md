# src/xrGame/script_effector.h

> Declares the script-authored post-process effector.

**Needs** — [`script_effector.cpp`](script_effector.cpp.md) · [`script_effector_inline.h`](script_effector_inline.h.md) · [`xrCore/PostProcess/PPInfo.hpp`](../xrCore/PostProcess/PPInfo.hpp.md) · [`xrEngine/Effector.h`](../xrEngine/Effector.h.md)
**Used by** — [`base_client_classes_script.cpp`](base_client_classes_script.cpp.md) · [`script_effector.cpp`](script_effector.cpp.md) · [`script_effector_inline.h`](script_effector_inline.h.md) · [`script_effector_script.cpp`](script_effector_script.cpp.md) · [`script_effector_wrapper.cpp`](script_effector_wrapper.cpp.md) · [`script_effector_wrapper.h`](script_effector_wrapper.h.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in [`script_effector.cpp`](script_effector.cpp.md), as a
specialization of the engine's post-process effector.

Exported units:

- `effector_type` — the slot identity, public because removal keys on it.
- `construct(type, duration)` — see [`script_effector_inline.h`](script_effector_inline.h.md).
- `process(parameters)` — the per-frame hook, in two forms; script overrides it.
- `add` / `remove` — attach to and detach from the actor's camera.
- A registration entry point that exports the class to the script layer.
