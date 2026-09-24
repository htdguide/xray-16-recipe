# src/xrEngine/ShadersExternalData.h

> The handful of values the game pushes into shader constants without going through the material system.

**Needs** — _none_
**Used by** — [`IGame_Persistent.h`](IGame_Persistent.h.md)
**Tier floor** — T1: the record is uploaded as shader constants, so its float layout is the interface

## Purpose

Almost everything a shader reads comes from the material description or from the renderer's
own per-frame constants. A few things do not: a matrix scripts can write freely, the
first-person weapon's zoom state, and which of several full-screen rendering modes is
active. Those are set from the game module at arbitrary times and read by the material
recorder when it binds constants, so they need one agreed place to live.

It is header-only and has no behaviour: it is a shared mailbox, and the whole file is the
statement of what is in it.

## State

```text
RECORD ShadersExternalData
  script_params : matrix4     # sixteen floats scripts may set for their own shaders
  hud_params    : real[4]     # [zoom_rotate_factor, second_viewport_zoom_factor, unused, unused]
  blender_mode  : real[4]     # x: main viewport mode, y: second viewport mode,
                              # z: unassigned, w: 0 = ordinary object, 1 = detail objects

# all fields start at zero except script_params, which starts as the identity matrix
```

`blender_mode`'s x and y select a full-screen vision mode — 0 ordinary, 1 night vision,
2 thermal, and further values the shared shader header defines. Two are needed because the
engine can render a second viewport (a scope, a camera feed) in a different mode from the
main one in the same frame. The w component is set by the renderer itself while it is
drawing the grass-and-debris layer, so a shader shared between ordinary geometry and detail
objects can tell which pass it is in.

**Invariants** — the record is written from the game and simulation phases and read during
the render phase. It is not synchronized: the two phases do not overlap, which is what
makes a bare shared record legal here. A rebuild that overlaps them must double-buffer it
per frame rather than lock it, because the render phase must see one consistent set of
values for the whole frame.

**Notes** — the vision-mode value is a float carrying an enumerated code because it is
uploaded as part of a four-component constant, and the constant slot is what is frozen.
The two "unassigned" slots exist because a four-component constant is the smallest unit
the device accepts.
