# src/xrGame/script_effector_wrapper.cpp

> Routes the effector's per-frame evaluation into the script object's `process` method.

**Needs** — [`script_effector_wrapper.h`](script_effector_wrapper.h.md) · [`script_effector.h`](script_effector.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: a call convention across the script boundary

## Purpose

The same two-function adapter pattern as
[`script_binder_object_wrapper.cpp`](script_binder_object_wrapper.cpp.md), applied to the
one overridable method an effector has.

## State

`Stateless.`

## `process(parameters) -> bool`

**Contract** — invokes the script object's `process` method with the parameter block and
returns its answer. The script mutates the block in place; the return value says whether
the effector should live another frame.

## `process_static(effector, parameters) -> bool`

**Contract** — invokes the *base* implementation directly, bypassing the script override.
This is what a script's `process` calls when it wants the inherited clock-advancing
behaviour; without the bypass it would re-enter its own override forever.
