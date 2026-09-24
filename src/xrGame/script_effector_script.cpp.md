# src/xrGame/script_effector_script.cpp

> Exports the post-process parameter block and the script effector to the script layer.

**Needs** — [`script_effector.h`](script_effector.h.md) · [`script_effector_wrapper.h`](script_effector_wrapper.h.md) · [`xrCore/PostProcess/PPInfo.hpp`](../xrCore/PostProcess/PPInfo.hpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: registration data only

## Purpose

Exports five types. Four of them are the post-process parameter block and its three
component records — this is the only place a script can read or write screen
post-processing, so the field names here are the whole vocabulary a mod has for it, and
they are frozen.

## `script_register`

**Contract** — registers, under these frozen script names:

```text
duality         h, v                      # horizontal and vertical double-vision offsets
color           r, g, b
noise           intensity, grain, fps     # film grain; fps is the grain's own update rate
effector_params blur, gray, dual, noise, color_base, color_gray, color_add
effector        constructed from (type, duration)
```

Each component record is default-constructible, constructible from its fields, and has a
`set` taking all its fields at once. `effector_params` additionally exposes `assign`, which
copies another block wholesale — scripts use it to snapshot and restore the frame's
parameters, which value assignment across the script boundary would otherwise not give
them.

The effector itself exposes:

```text
start    -> add       # ownership of the effector passes to the camera
finish   -> remove    # ownership of the effector passes to the camera
process  -> process   # overridable; paired with the base implementation
```

**Notes**

`start` and `finish` are both marked as transferring ownership of the effector to the
engine. That is correct for `start` and surprising for `finish`: after removal the script's
handle must not be reused either, because removal destroys whatever occupied the slot.
Naming the script-side field `dual` while the engine field is `duality` is a frozen
inconsistency.
