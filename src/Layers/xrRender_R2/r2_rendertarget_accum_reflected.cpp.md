# src/Layers/xrRender_R2/r2_rendertarget_accum_reflected.cpp

> Adds one indirect-bounce light into the accumulator — like a point light, but with no
> stencil bound of its own.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`r2_types.h`](r2_types.h.md) · [`r2_rendertarget_phase_accumulator.cpp`](r2_rendertarget_phase_accumulator.cpp.md) · [`r2_rendertarget_draw_volume.cpp`](r2_rendertarget_draw_volume.cpp.md) · [`r2_rendertarget_enable_scissor.cpp`](r2_rendertarget_enable_scissor.cpp.md) · [`xrRender/light.h`](../xrRender/light.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: stencil comparisons and matrix construction against clip-space
conventions.

## Purpose

A bounce light is a synthetic light standing for the diffuse reflection off a surface
that a real light illuminated (see [`r2_R_lights.cpp`](r2_R_lights.cpp.md), where they
are created). Many of them are accumulated per real light, so each one must be as cheap as
possible. The saving is that it skips the two volume passes that mark the stencil, and
lights every covered pixel inside its sphere instead — accepting some overdraw in exchange
for a third of the draws.

## `accumulate_reflected_light`

**Contract** — adds the bounce's contribution to the accumulator. Binds the accumulator
first. Does not advance the light marker, because it never claimed one.

```text
FUNCTION accumulate_reflected_light(light)
  bind the accumulator; count it as visible
  world = the light's transform; view and projection = the camera's
  back_faces_needed = does the light's sphere reach the near plane
  offer the depth-bounds hint
  cull back faces if so, front faces otherwise      # the point path ignores this; here
                                                    # it is honoured
  texgen = screen coordinates, with a half-texel offset folded in
  colour, specular = the bounce's colour and its derived gloss scale
  eye_position, eye_direction = the bounce's position and direction, in view space

  material = the reflected description
  constants: (eye position, 1/range^2), (colour, specular),
             (eye direction, 0), the texgen matrix
  where stencil >= 1: draw the sphere
  IF multisampling: repeat per sample where the edge bit is set
  blend-copy if the accumulator cannot be blended into
  release the depth-bounds hint
```

**Invariants** — the stencil test is "at least one", meaning "a surface is here", rather
than "equal to a marker". That is the entire difference from a real light, and it is only
safe because the sphere is convex and the light's falloff reaches zero at its surface: a
pixel outside the sphere but inside its screen-space silhouette receives a contribution of
zero rather than a wrong one.

**Notes** — the overdraw this accepts is real. A bounce sphere seen edge-on covers the
whole depth range behind it, and every covered pixel in its silhouette runs the lighting
program. It is affordable because the program is short — no shadow lookup, no projector —
and because the bounce ranges are small by construction (see the range derivation in
[`r2_R_lights.cpp`](r2_R_lights.cpp.md)).

The half-texel offset folded into the texture-coordinate matrix here, and absent from the
point light's, compensates for the difference between pixel centres and texel corners on
one backend. It is convention plumbing, not a decision.

Bounce lights feed the accumulator through the same blend-copy tail as everything else on
devices without float blending, and through the same per-sample repeat under
multisampling. Neither is specific to them.
