# src/xrGame/script_effector_wrapper.h

> Declares the script-override adapter for the post-process effector.

**Needs** — [`script_effector.h`](script_effector.h.md) · [`script_effector_wrapper_inline.h`](script_effector_wrapper_inline.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_effector_script.cpp`](script_effector_script.cpp.md) · [`script_effector_wrapper.cpp`](script_effector_wrapper.cpp.md) · [`script_effector_wrapper_inline.h`](script_effector_wrapper_inline.h.md)
**Tier floor** — T2: a declaration

## Purpose

Declares the surface implemented in
[`script_effector_wrapper.cpp`](script_effector_wrapper.cpp.md): the overriding form of
`process` that dispatches into script, and the static form that reaches the base
implementation. The constructor is in
[`script_effector_wrapper_inline.h`](script_effector_wrapper_inline.h.md) and forwards
unchanged.
