# src/xrPhysics/PHElementNetState.cpp

> One rigid body's complete simulation state, in and out of a network packet — including the previous interpolation sample, so the receiver can render smoothly from the first packet.

**Needs** — [`PHElement.h`](PHElement.h.md) · [`PHInterpolation.h`](PHInterpolation.h.md) · [`PHShell.h`](PHShell.h.md) · [`PHDisabling.h`](PHDisabling.h.md) · [`xrServerEntities/PHSynchronize.h`](../xrServerEntities/PHSynchronize.h.md) · [Seam: Rigid-body dynamics](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: it moves a fixed record of numbers across a boundary; the bit-packing itself lives in the network layer.

## Purpose

A separate file for four short procedures, because the state a body carries across the wire is a
*decision* and not an implementation detail: it fixes what the receiving end can reconstruct and
what it must guess. A rebuild may merge this back into the element.

## State

```text
RECORD ElementNetState              # the wire-visible state of one body
  position             : vector     # the DYNAMIC placement, not the interpolated render one
  quaternion           : rotation
  previous_position    : vector     # interpolation sample 0
  previous_quaternion  : rotation
  linear_vel           : vector
  angular_vel          : vector
  force                : vector     # accumulated, not yet applied
  torque               : vector
  enabled              : bool       # is the body awake
```

**Invariants** — position and quaternion are the *solved* state, never the render-interpolated one:
a receiver that imported an interpolated pose would be a fraction of a step behind and would drift.
An inactive element exports `enabled = false` regardless of anything else.

## `get_State`

**Contract** — fills the record from the live body. Reads the dynamic (not interpolated) placement,
the current velocities, the accumulated force and torque, and interpolation sample 0 as the
previous pose. Does not allocate, does not modify the body. An element that is not active reports
`enabled = false` and stops there, leaving the rest of the record as whatever the getters produced.

**Notes** — exporting the *accumulated* force and torque as well as the velocities is what makes the
handover exact: the sender may be mid-frame, with forces applied but not yet integrated, and a
receiver that took only velocity would integrate a different step. This is the same reason the
[determinism requirement](../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) exists — both ends
step the same world and compare.

## `set_State`

**Contract** — installs the record onto the live body and rebuilds the interpolation window from it.
Marks the element as needing a render update. Changes the body's sleep state to match, routing that
change through the *shell* so the whole shell's island wakes or sleeps together. Resets the sleep
accumulator so an imported body does not immediately fall asleep on the receiver's stale history.

```text
FUNCTION set_state(state)
  needs_render_update := true
  set the dynamic position and orientation from the state
  interpolation sample 0 := (previous_position, previous_quaternion)
  interpolation sample 1 := (position, quaternion)       # the window is now complete
  velocities, force and torque := the state's
  IF NOT active THEN RETURN
  IF state.enabled AND body is asleep THEN wake the body ; shell.enable_object()
  IF NOT state.enabled AND body is awake THEN shell.disable_object() ; sleep the body
  reset the sleep accumulator
```

**Invariants** — both interpolation samples are written, not just the current one. A receiver with
one sample has no window to blend across and renders a step of stutter on every packet; this is why
the previous pose travels on the wire at all rather than being recovered from velocity.

**Notes** — waking goes through the shell and not the body, because sleep in this engine is an
island-level property: waking one body of a multi-body object while its neighbours sleep leaves the
solver with a half-active constraint set and the object visibly tears. See
[`PHIsland.cpp`](PHIsland.cpp.md).

## `net_Export` / `net_Import`

**Contract** — pure adapters: build the record, hand it to the packet's own serializer, and the
reverse. The quantization, bit widths and ordering are the network layer's business, not this
file's — see [`xrServerEntities/PHSynchronize.h`](../xrServerEntities/PHSynchronize.h.md).
