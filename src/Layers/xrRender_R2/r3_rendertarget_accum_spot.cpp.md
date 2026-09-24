# src/Layers/xrRender_R2/r3_rendertarget_accum_spot.cpp

> Adds one spot light into the accumulator, including the matrix that turns a G-buffer
> pixel into an atlas lookup — and, separately, the slice stack that draws its light shaft.

**Needs** — [`r2_rendertarget.cpp`](r2_rendertarget.cpp.md) · [`r2_types.h`](r2_types.h.md) · [`r2_rendertarget_phase_accumulator.cpp`](r2_rendertarget_phase_accumulator.cpp.md) · [`r2_rendertarget_draw_volume.cpp`](r2_rendertarget_draw_volume.cpp.md) · [`r2_rendertarget_accum_spot_geom.cpp`](r2_rendertarget_accum_spot_geom.cpp.md) · [`xrRender/light.h`](../xrRender/light.h.md) · [`xrRender/du_cone.h`](../xrRender/du_cone.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: matrix construction against device clip-space conventions, and
stencil masks.

## Purpose

Spot accumulation is point accumulation plus a shadow lookup, and the shadow lookup is the
part worth reading: it is where a light's atlas rectangle, the depth bias and the two
backends' clip conventions all meet. The second entry point in the file is unrelated in
shape — it draws the light's volumetric shaft as a stack of camera-facing slices.

## `accumulate_spot_light`

**Contract** — adds the light's contribution. Identical in structure to
[the point light](r3_rendertarget_accum_point.cpp.md): bind the accumulator, bound the
light's pixels with front and back volume passes writing the light marker, then run the
lighting program where the stencil equals the marker, with a per-sample repeat under
multisampling. Two differences: the volume is a cone (or a sphere cap for one face of a
cube-shadowed light, which routes here too), and two extra matrices are supplied.

```text
FUNCTION accumulate_spot_light(light)
  material = the light's own description, or the default spot one
             (or the point one, for a cube-face sub-light)
  ... the stencil bound, exactly as for a point light ...
  build the shadow matrix and the projector matrix (below)
  colour  = light colour scaled by the light's distance-based dimming factor
  radius  = light.range * 0.95
  constants: (eye position, 1/radius^2), (colour, specular),
             the screen texgen, the jitter texgen,
             the shadow matrix, and the projector matrix's first two rows
  element = shadowed ? (full-atlas ? FULLSIZE : translucent ? TRANSLUENT : NORMAL)
                     : UNSHADOWED, and in the unshadowed case the shadow matrix is
                     replaced by the projector matrix
  draw the cone where stencil == marker; repeat per sample on edges
  blend-copy if the accumulator cannot be blended into
  advance the light marker
```

### The shadow matrix

**Contract** — maps a point in view space to a coordinate in the light's atlas rectangle
plus a biased depth. Built per light, per frame.

```text
FUNCTION build_shadow_matrix(light) -> matrix
  side    = the atlas's side in texels
  texel   = 0.5 / side                       # half-texel centring
  extent  = (light.rect.side - 2) / side     # note: minus two
  origin  = (light.rect.x + 1) / side, (light.rect.y + 1) / side
  depth_scale = the configured depth scale ; depth_bias = the configured bias

  adjust = scale x,y by extent/2 and translate to extent/2 + origin + texel
           scale z by depth_scale and translate by depth_bias
           # one backend flips y here and maps z into [0,1] rather than [-1,1]

  RETURN inverse_view * light.view * (adjust * light.projection)
```

**Invariants** — the one-texel inset on every side (the `+1` on the origin and the `-2` on
the extent) is mandatory. Shadow lookups filter, and without the inset a filter tap at a
rectangle's edge reads its neighbour's depth, which shows as a bright or dark seam
crossing the shadow. The inset is also why the allocator refuses rectangles smaller than
five texels.

**Notes** — the matrix chain reads right to left: undo the camera's view to get world
space, apply the light's view and projection to get its clip space, then the adjustment to
land in atlas coordinates. Composing it once per light rather than per pixel is the point;
the program receives one matrix and does one transform.

The *projector* matrix is the same construction with the rectangle set to the whole
texture. It is used for the light's colour mask — a spot light may project a texture, a
gobo — and it doubles as the shadow matrix for unshadowed lights so that the program does
not need a separate code path. Only its first two rows are sent, because the mask lookup
needs only two coordinates.

The depth scale and bias are user settings rather than derived. They are the classic
shadow-acne knob and a rebuild will need its own; the mechanism to preserve is that they
are folded into the *matrix*, not applied in the program, so they cost nothing per pixel.

## `accumulate_volumetric`

**Contract** — draws the light's shaft into the separate volumetric target, as a stack of
camera-facing slices blended additively. Does nothing for a light not marked volumetric.
Does not touch the stencil or the light marker.

```text
FUNCTION accumulate_volumetric(light)
  IF the light is not volumetric THEN RETURN
  bind the volumetric accumulator
  no culling; colour writes on (RGB only)
  build the same shadow and projector matrices as above

  # --- the clipping frustum ---------------------------------------------------
  frustum = the light's own frustum, in world space
  pull its far plane in toward the near plane by (1 - volumetric_distance)
      # the shaft may be shorter than the light's reach
  transform all six planes into clip space and hand them to the program

  # --- the bounding box the slices are stretched over --------------------------
  scaled_radius = light.sphere.radius * volumetric_distance
  centre = the light's sphere centre, pulled toward the light by the same factor
  box = that centre and radius, in view space

  # --- quality: fewer slices, each carrying more energy -----------------------
  slices  = max(10, slice_capacity * quality)
  quality = slices / slice_capacity               # re-derived, so it is exact
  colour  = light colour * intensity * distance * (1/quality) * dimming
  radius  = volumetric_distance * light.range * 0.95

  constants: (eye position, 1/radius^2), (colour, specular), both texgens,
             the shadow and projector matrices,
             the box's minimum, and its maximum with the far z stretched by 1/quality
  draw the whole slice stack
  clear the clip planes
```

**Invariants** — the far bound of the box is stretched by the reciprocal of the quality
*and* the colour is scaled by the same reciprocal. Together these keep the integrated
brightness of the shaft constant as the slice count drops: fewer, thicker slices, each
proportionally brighter. Getting one without the other makes the quality setting change
the shaft's brightness rather than its smoothness.

**Notes** — the slice *capacity* is drawn in full regardless of the computed count. The
count is computed, used to scale the energy, and then ignored by the draw — the whole
stack is always issued. Whether that is intentional (the program discards slices outside
the box, so a shorter draw would be a pure saving) or a regression is not recoverable from
the source; the energy compensation only makes sense if the shorter draw was meant.

Six user clip planes rather than a stencil bound: the shaft is not bounded by geometry, so
there is nothing for a stencil to mark. The planes are transposed through the inverse
combined transform and negated to reach clip space, which is the standard way to clip in
post-projective space.
