# src/Layers/xrRender_R2/r3_rendertarget_accum_point.cpp

> Adds one point light into the accumulator: bound its pixels with the stencil, then run
> the lighting program over exactly those.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`r2_types.h`](r2_types.h.md) · [`r2_rendertarget_phase_accumulator.cpp`](r2_rendertarget_phase_accumulator.cpp.md) · [`r2_rendertarget_draw_volume.cpp`](r2_rendertarget_draw_volume.cpp.md) · [`r2_rendertarget_enable_scissor.cpp`](r2_rendertarget_enable_scissor.cpp.md) · [`xrRender/light.h`](../xrRender/light.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: stencil operations whose exact comparison and write masks are the
algorithm.

## Purpose

The canonical light accumulation, and the clearest statement of the stencil trick the
whole lighting pass rests on. A spot light
([`r3_rendertarget_accum_spot.cpp`](r3_rendertarget_accum_spot.cpp.md)) is this with a
different volume and a shadow lookup; an indirect bounce
([`r2_rendertarget_accum_reflected.cpp`](r2_rendertarget_accum_reflected.cpp.md)) is this
without the stencil bound.

## `accumulate_point_light`

**Contract** — adds the light's contribution to the accumulator. Binds the accumulator
first (clearing it if this is the frame's first light). Leaves the light marker advanced.
Requires the light's transform and, if it casts shadows, its atlas rectangle to be set.

```text
FUNCTION accumulate_point_light(light)
  bind the accumulator
  material = the light's own accumulation description, or the default point one
  effective_radius = light.range * 0.95          # see notes
  eye_position = the light's position in view space
  colour, specular = the light's colour and its derived gloss scale
  world = the light's transform; view and projection = the camera's
  back_faces_needed = does the light's sphere reach the near plane
  offer the depth-bounds hint

  # ---- bound the light's pixels in the stencil ---------------------------------
  material element = the mask description's point element    # colour writes off
  cull front faces:  where stencil >= 1 and depth FAILS, write the light's marker
  draw the volume
  cull back faces:   where stencil >= 1 and depth FAILS, write 1
  draw the volume
  # what survives holding the marker is exactly the pixels between the volume's
  # front and back surfaces that contain geometry

  # ---- light them ----------------------------------------------------------------
  colour writes on; cull front faces
  texgen  = screen texture coordinates from the volume's clip position
  jitter  = the same, tiled to the noise texture
  element = shadowed ? (full-atlas ? FULLSIZE : (translucent ? TRANSLUENT : NORMAL))
                     : UNSHADOWED
  constants: (eye position, 1/radius^2), (colour, specular), the texgen matrix
  where stencil == the marker: draw the volume

  IF multisampling
      then again with the per-sample variant where stencil == marker | edge bit,
      either once (device selects the sample) or once per sample with a sample mask

  # ---- blend-copy, only where float blending is unavailable --------------------
  IF the accumulator cannot be blended into
      bind the real accumulator
      element = the mask description's volumetric-accumulation element
      where stencil selects this light: draw the volume again, copying the twin across

  advance the light marker
```

### Why the stencil dance works

The volume is convex and closed. For a pixel *inside* it, the volume's back surface is
behind the scene geometry (its depth test fails) and its front surface is in front (its
depth test passes). For a pixel outside, either both fail or both pass. So: writing the
marker where the back faces fail depth claims every pixel at or beyond the volume's far
surface, and then writing one where the front faces fail depth releases every pixel beyond
the volume's near surface. The difference — pixels holding the marker — is the inside.

This is the classical depth-fail shadow-volume counting argument, applied to a light
instead of a shadow. Two properties are assumed and are true here: the volume is convex,
and no two lights' volumes are processed concurrently. Convexity is why one front pass and
one back pass suffice instead of a counter.

**Invariants** — the comparison is "greater than or equal to one" on the stencil, not
"equal", because the value there may be a previous light's marker; the *write* is
unconditional on the depth-fail branch. Under multisampling the read and write masks drop
the high bit, so edge markings survive. The marker itself is always odd (see
[`r2_rendertarget.cpp`](r2_rendertarget.cpp.md)), which is what lets the "at least one"
test and the "exactly this light" test coexist.

**Notes** — the radius is scaled to 95 percent of the light's range before being handed to
the program as an inverse square. The volume mesh is built around the full range, so the
attenuation reaches zero slightly *inside* the mesh; without the margin the mesh's own
silhouette would be visible as a hard circular edge wherever the falloff had not quite
reached zero at the boundary.

Front faces are drawn for the lighting pass regardless of the near-plane answer — the
point path computes the answer and then ignores it, with the alternative left in the
source as a comment. The spot path does use it. That asymmetry looks like an oversight and
may be one; a point light whose sphere contains the camera will have its front faces
clipped, and the fix is the same one the spot path already applies.
