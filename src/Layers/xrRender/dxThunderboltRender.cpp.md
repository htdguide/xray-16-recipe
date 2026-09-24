# src/Layers/xrRender/dxThunderboltRender.cpp

> Draws one lightning strike: the bolt mesh stamped into the shared dynamic buffers with an animated texture shift, plus two camera-facing glow quads whose brightness tracks the flash.

**Needs** — [`Include/xrRender/ThunderboltRender.h`](../../Include/xrRender/ThunderboltRender.h.md) · [`dxThunderboltRender.h`](dxThunderboltRender.h.md) · [`dxThunderboltDescRender.h`](dxThunderboltDescRender.h.md) · [`Include/xrRender/LensFlareRender.h`](../../Include/xrRender/LensFlareRender.h.md) · [`xrEngine/thunderbolt.h`](../../xrEngine/thunderbolt.h.md) · [`Include/xrRender/RenderDetailModel.h`](../../Include/xrRender/RenderDetailModel.h.md) · [`FVF.h`](FVF.h.md) · [`R_Backend.h`](R_Backend.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dxThunderboltRender.h`](dxThunderboltRender.h.md)
**Tier floor** — T1: it writes both vertices and indices into mapped device buffers at declared strides, and it overrides depth state directly because the material it borrows has the wrong one.

## Purpose

The engine's thunderbolt effect owns when a strike happens, where it is, how long the flash lasts and which description was chosen. This file owns the three draws that make it visible. Its interesting content is not the drawing but two decisions: how the bolt's texture is *animated* without any per-frame texture work, and why the glow quads have to fight the material they borrow.

## State

```text
RECORD ThunderboltRenderer
  bolt_geometry   : GeometryDecl  # position + colour + one uv; shared dynamic vertex
                                  # AND index streams, because the mesh has real indices
  glow_geometry   : GeometryDecl  # position + colour + one uv, already in clip space;
                                  # shared dynamic vertex stream against the shared quad
                                  # index buffer
```

Both are created at construction and released at teardown. Neither owns a material: the bolt borrows the mesh's, and each glow quad borrows the material of the lens-flare object the description points at.

## `render`

**Contract** — Draw the currently active strike. Requires that the owner has one — there is no "nothing to draw" path; the engine calls this only while a strike is live. Reads the owner's flash phase, size, transform and centre; reads the chosen description's mesh and its two gradient sprites. Writes into the shared dynamic streams and issues three draws. Leaves face culling as it found it; deliberately leaves depth state changed on the backend that needs it (see Notes).

```text
FUNCTION render(effect)
  # ---- 1. The bolt mesh -------------------------------------------------
  # Texture animation without touching a texture: the mesh's loader supports a
  # per-copy shift of the texture coordinates, and the bolt's texture holds two
  # variants side by side. For the first half of the flash the shift ramps
  # smoothly from 0 to 0.25; after the halfway point it snaps to one of two
  # discrete values chosen at random each frame, which reads as the flicker of
  # a dying strike rather than a fade.
  IF effect.phase > 0.5
    shift = random_choice_of(0.0, 0.5)
  ELSE
    shift = effect.phase * 0.5

  disable face culling          # the bolt mesh is a flat ribbon, visible from both sides
  mesh = effect.current_description.render_half.model
  vbuf = map_vertices(mesh.vertex_count, bolt_geometry.stride)
  ibuf = map_indices(mesh.index_count)
  stamp mesh INTO (vbuf, ibuf) WITH transform = effect.current_transform,
                                    colour    = opaque white,
                                    uv_shift  = shift
  unmap both
  set world transform = identity        # the transform is already baked into the vertices
  set material = mesh.material
  draw_indexed_triangles(bolt_geometry, vbuf, ibuf)
  restore face culling

  # ---- 2. The two glow quads -------------------------------------------
  # Eight vertices in one map: four for the "top" gradient, anchored at the
  # strike's origin, and four for the "centre" gradient, anchored at the
  # effect's own centre point. Both are billboards built from the camera's
  # right and up axes, so they face the viewer whatever the strike's rotation.
  buf = map_vertices(8, glow_geometry.stride)
  FOR EACH gradient IN (top at effect.transform.position,
                        centre at effect.centre)
    level  = floor(gradient.opacity * effect.phase * 255)
    colour = grey(level) with alpha = level     # brightness AND alpha track the flash
    dx = camera_right * gradient.radius.x * effect.size
    dy = camera_up    * -gradient.radius.y * effect.size   # negated: uv v grows downward
    emit anchor + dx - dy  uv (0,0)
    emit anchor + dx + dy  uv (0,1)
    emit anchor - dx - dy  uv (1,0)
    emit anchor - dx + dy  uv (1,1)
  unmap

  set world transform = identity
  FOR EACH gradient, quad_index IN (top -> 0, centre -> 4)
    set material = gradient.flare.material
    force depth test ON with a less-or-equal comparison     # see Notes
    draw_two_triangles(glow_geometry, buf + quad_index)
```

**Invariants**

- The eight glow vertices are written in the *quad order* the shared index buffer expects (`0,1,2 / 3,2,1` per four), which is why each quad is emitted as `+dx-dy, +dx+dy, -dx-dy, -dx+dy` and not as a ring. Changing the order silently turns the quad inside out.
- The bolt's world transform is set to identity because the mesh stamp already transformed every vertex. Setting it twice would double the rotation.

**Notes**

- The glow quads borrow the *sun flare's* material, because a lightning glow and the sun's glare want the same additive, textured, camera-facing behaviour. That material is authored for the sun, which is drawn with depth writing disabled and no depth test — correct for something at infinity, wrong for a bolt that is a few hundred metres away and must be occluded by terrain. The renderer therefore overrides depth state to *on, less-or-equal* immediately before each glow draw, after the material has been applied. This is a real dependency on being able to override a material's state at draw time; a rebuild that makes materials immutable state objects must instead give lightning its own material.
- That override is applied only on the Direct3D path in the original, with a note questioning whether the OpenGL path needs it. Treat this as unresolved: a rebuild should apply it unconditionally, since the sun material's depth settings are a property of the material, not of the API.
- Neither the flash brightness curve nor the two discrete flicker values are derived from anything; they are authored constants and must be reproduced as written for the effect to look the same.

## `copy`

**Contract** — Adopt another bolt renderer's contents field-wise. The geometry declarations are shared references, not clones.
