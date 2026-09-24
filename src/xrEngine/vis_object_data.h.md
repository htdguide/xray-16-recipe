# src/xrEngine/vis_object_data.h

> The per-visual parameters handed to material passes — camouflage, "living object" readouts, and two model-specific overrides that have nothing to do with rendering.

**Needs** — _(none beyond the math vocabulary)_
**Used by** — [`vis_common.h`](vis_common.h.md)
**Tier floor** — T1: the vector and matrix members are uploaded to the graphics device as constant-buffer contents, so their component order and width are fixed by the shader side.

## Purpose

A material pass is shared by every object that uses it, but several effects need per-object
numbers: a camouflage transform, a thermal-vision intensity, a health readout that tints a
creature. This record is where those live, one per visual (or one shared by a group of
visuals — see [`vis_common.h`](vis_common.h.md)).

Two unrelated fields ride along because they are also per-model and there was nowhere else
to put them; the file says so, and a rebuild is free to move them.

## State

```text
RECORD ObjectShaderData
  max_bullet_bones : int             # default 0
  hud_custom_fov   : real            # default -1 meaning "use the global HUD field of view"

  camo_transform   : 4x4 matrix      # to the material pass
  custom_params    : 4 reals         # meaning is agreed between one material and one caller
  entity_params    : 4 reals         # health, radiation, condition, thermal intensity
```

Invariants:

- `entity_params` uses **-2 as "this object has no such property"** for the first three
  components, and defaults to (-2, -2, -2, 0). The sentinel is negative and outside the
  valid range of all three (which are fractions in 0..1), so a pass can branch on it without
  a separate flag. The fourth component is a plain 0..1 intensity with no sentinel.
- `hud_custom_fov` uses **-1 as "unset"** for the same reason: a field of view is positive.
- `custom_params` has no fixed meaning. It is a deliberately untyped channel between one
  material pass and whatever code sets it, and it is therefore the one field a rebuild
  cannot document — the contract lives in the shipped shader sources.

## Notes

`max_bullet_bones` is the count of bones in a model that are bound to the remaining
ammunition, so a belt or magazine visibly empties. Storing it here means the game layer
sets it once at load and the animation layer reads it per frame.

The record is constructed with its defaults rather than zeroed, which matters: a zeroed
`entity_params` would read as "health zero, radiation zero, condition zero" — a dead,
pristine object — instead of "not applicable".
