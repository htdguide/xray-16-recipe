# src/xrEngine/vis_common.h

> The per-visual visibility record — the bounds a culler tests, and the frame stamps that stop it being tested twice.

**Needs** — [`vis_object_data.h`](vis_object_data.h.md) · [`xrCore/_sphere.h`](../xrCore/_sphere.h.md) · [`xrCore/_fbox.h`](../xrCore/_fbox.h.md)
**Used by** — [`RenderVisual.h`](../Include/xrRender/RenderVisual.h.md) · [`FBasicVisual.h`](../Layers/xrRender/FBasicVisual.h.md) · [`Render.h`](Render.h.md)
**Tier floor** — T1: the record is embedded by value in every renderable visual and its size is a per-object memory cost multiplied by thousands; the original fixes its packing explicitly.

## Purpose

Every visual carries this block, and the visibility pass reads nothing else about an object
before deciding whether to draw it. It answers two questions: *where is this thing* (a
sphere and a box in world space) and *has the culler already dealt with it this frame*.

## State

```text
RECORD VisibilityData
  sphere       : (centre, radius)     # cheap reject
  box          : axis-aligned bounds  # precise reject
  marker       : list<int>            # one frame stamp per render context
  accept_frame : int                  # frame this visual was admitted to the main render
  hom_frame    : int                  # frame the occlusion test is next scheduled for
  hom_tested   : int                  # frame the occlusion test last ran
  shader_data  : ObjectShaderData     # see vis_object_data.h; defaults to the own copy
```

Invariants:

- Sphere and box describe the same volume in the same space; the sphere is the conservative
  one and is tested first. Cleared state is a zero-radius sphere at the origin and an
  *invalid* box — invalid meaning "no points", so that growing it by the first real point
  produces exactly that point rather than a box reaching back to the origin.
- `marker` has one slot per render context. The renderer runs a fixed number of parallel
  scene-graph contexts plus one immediate context, and each walks the scene
  independently; a single shared stamp would make one context's visit hide the object from
  another's. The array length is therefore *parallel contexts + 1*, and that "+1" is the
  immediate context.
- `shader_data` points at the embedded own-copy unless something has redirected it. A
  visual that shares another's shader parameters — the parts of a composite model, a
  weapon's world and HUD variants — points them all at one record so a change is made once.

## Notes

The occlusion pair (`hom_frame`, `hom_tested`) is a *schedule*, not a result: testing every
visual against the occlusion buffer every frame costs more than it saves, so each visual
carries the frame it is next due and the frame it was last done. The decision recorded here
is that occlusion culling is rate-limited per object, independent of the frustum culling
that is not.

Carrying a full shader-parameter record inside every visual is acknowledged in the source
as wasteful — it is per-model data stored per-instance-part. It is kept because the
alternative is an indirection on a path that runs for every drawn thing. A rebuild with a
side table keyed by visual pays a lookup instead of the memory.
