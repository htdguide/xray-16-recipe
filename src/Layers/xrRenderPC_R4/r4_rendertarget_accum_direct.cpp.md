# src/Layers/xrRenderPC_R4/r4_rendertarget_accum_direct.cpp

> Adding the sun to the lighting accumulator: how each cascade claims its band of the screen, how the shadow lookup is built, how the two sun implementations differ, and how light shafts ride along.

**Needs** — [`r4_rendertarget.h`](r4_rendertarget.h.md) · [`r4_rendertarget_u_set_rt.cpp`](r4_rendertarget_u_set_rt.cpp.md) · [`../xrRender_R2/r2_rendertarget_phase_accumulator.cpp`](../xrRender_R2/r2_rendertarget_phase_accumulator.cpp.md) · [`../xrRender_R2/render_phase_sun.cpp`](../xrRender_R2/render_phase_sun.cpp.md) · [`../xrRender_R2/render_phase_sun_old.cpp`](../xrRender_R2/render_phase_sun_old.cpp.md) · [`../xrRender_R2/r3_rendertarget_create_minmaxSM.cpp`](../xrRender_R2/r3_rendertarget_create_minmaxSM.cpp.md) · [`../xrRender_R2/r2_types.h`](../xrRender_R2/r2_types.h.md) · [`../xrRenderDX11/dx11R_Backend_Runtime.h`](../xrRenderDX11/dx11R_Backend_Runtime.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: stencil semantics, depth-comparison choice and per-sample masking are the whole content, and each is a device-level decision.

## Purpose

The sun is the one light that covers the entire screen, and the one whose shadow map cannot
be a single texture — it is split into cascades, each a shadow map fitted to a slab of the
view frustum. Accumulating it is therefore not "draw the light volume" but "for each
cascade, work out which screen pixels belong to it, and add the sun there with that
cascade's shadow lookup".

Two implementations of the sun ship (see
[`../xrRender_R2/README.md`](../xrRender_R2/README.md) — which one runs is decided by
inspecting the installed game data's shader source), and this file serves both: the older
two-region sun calls `accum_direct`, the newer three-cascade sun calls
`accum_direct_cascade`. They share the masking step, the shadow matrix and the multisample
shape; they differ entirely in **how a cascade claims its pixels**, which is the interesting
part and is set out under the second heading.

The file is long because the sun has five sub-phases and three quality branches that
multiply. It is not five decisions per branch — it is the same handful of decisions,
restated. Those decisions are what follows.

## State

Two constant tables and one accumulating value.

```text
# the corners of a cascade's normalized clip box, to be transformed back to world space
CASCADE_BOX_CORNERS : 8 points, x and y in {-1, +1}, z in {0.7, 1.0}
# the twelve triangles closing that box; four further slots exist and are degenerate
CASCADE_BOX_FACES   : 16 triples of corner indices, of which 12 are real

cloud_drift : real   # persists across frames; advances by 0.003 per second, forever
```

**Invariants** — the box spans only the far part of the cascade's normalized depth range,
from 0.7 to 1.0, not the whole of it. **Unrecovered**: nothing in the source explains 0.7,
and it is not derived from any configured split distance.

`cloud_drift` is an unbounded accumulator with no wrap. Over a long session it grows large
enough that the single-precision texture coordinate it feeds loses resolution and the cloud
shadows visibly quantize. A rebuild should wrap it at the cloud texture's period.

## `accum_direct` — the legacy sun

**Contract** — adds one sub-phase of the two-region sun into the lighting accumulator.
Sub-phases are *near*, *far* and *luminance*. Binds the accumulator, masks on the first
sub-phase only, then draws one full-screen quad with the sun material. Delegates to the
filtered path when the sun-filter option is on. Tail-calls the light-shaft pass on the far
sub-phase. Does not advance the light marker — `accum_direct_blend` does.

```text
FUNCTION accumulate_sun(cmd_list, sub_phase)
  bind the accumulator
  IF the sun-filter option is on THEN RETURN accumulate_sun_filtered(cmd_list, sub_phase)

  # the near sub-phase has a cheaper program when a min/max shadow pyramid exists
  element := sub_phase
  IF sub_phase = NEAR AND a min/max pyramid was built this frame
      element := NEAR_MINMAX

  sun_colour    := the sun light's colour
  sun_specular  := the scalar specular weight derived from that colour
  sun_direction := the sun's direction transformed into eye space, normalized

  IF sub_phase = NEAR THEN mask_sunlit_pixels(cmd_list, sun_colour, sun_direction)

  # the quad's depth is the split plane, so the device clips the quad
  # at the boundary between this region and the next
  quad_depth := clip-space z of a point at the near-split distance along the camera

  bind the accumulator again ; cull nothing ; colour writes on
  shadow_lookup := build_shadow_lookup(sub_phase, near_bias negated, far_bias)
  cloud_lookup  := build_cloud_lookup()
  emit a full-screen quad carrying screen coordinates and jitter coordinates
  material := the sun description, element `element`
  constants: sun_direction, sun_colour with sun_specular, shadow_lookup, cloud_lookup
  draw, per the multisample shape
  IF advanced post-processing is on AND light shafts are enabled AND sub_phase = FAR
      accumulate_light_shafts(cmd_list, sub_phase, the quad just emitted,
                              shadow_lookup, 0, the far split distance)
```

**Invariants** — the near region and the far region are separated by the **quad's own
depth**, not by a stencil or a depth comparison the pass chooses. Both regions draw the same
full-screen quad; the near one places it at the split plane and relies on the device's depth
test to reject the pixels beyond it. That is cheap and it is why the older sun needed no box
geometry — and it is also why it supports only two regions, since a quad has one depth.

## `mask_sunlit_pixels` — the masking step

Run once per frame, on the first sub-phase, in all three paths.

**Contract** — writes the current light marker into the stencil of every pixel that could
receive sunlight, and only those. One full-screen quad with the mask material, colour writes
off.

```text
FUNCTION mask_sunlit_pixels(cmd_list, sun_colour, sun_direction)
  # a perceptual luminance, not an average
  intensity := 0.30*red + 0.48*green + 0.22*blue
  # the program compares the surface normal against this vector and rejects
  # what faces away; the vector's *length* is the rejection threshold
  masking_vector := -sun_direction * sqrt(intensity)
  material := the mask description, its DIRECT element
  stencil: where the stored value is at least 1, replace it with the light marker
  draw, per the multisample shape
```

**Invariants** — the threshold scales with the **square root** of the sun's luminance. A
surface is admitted when its normal's projection onto the sun direction exceeds the
threshold, so a bright sun admits surfaces that are nearly edge-on to it and a dim one does
not. This is a cost optimization with a visual side effect: at dawn and dusk the sun stops
being evaluated on grazing surfaces slightly before it stops contributing, which is
invisible because its contribution there is already negligible. A rebuild that skips the
mask entirely is correct and slower.

The stencil predicate is *at least 1*, and 1 is what the G-buffer fill wrote on every
covered pixel. Sky pixels have 0 and are never sunlit.

## `build_shadow_lookup` — from world space to a shadow-map sample

Every sun path builds the same operator and hands it to the program as one matrix.

```text
FUNCTION build_shadow_lookup(sub_phase, bias) -> matrix
  depth_scale := near_depth_scale when sub_phase = NEAR, else far_depth_scale
  # map a clip position to a texture coordinate: halve x, halve and flip y,
  # scale and bias z into the range the shadow map stored
  adjust := [ 0.5   0     0            0
              0    -0.5   0            0
              0     0     depth_scale  0
              0.5   0.5   bias         1 ]
  lookup := adjust composed with this cascade's combined light transform
                   composed with the inverse view
  IF sub_phase = FAR AND the device does hardware shadow comparison
      # the far region's projection is trapezoidal, which stretches texels
      # unevenly; shift the receiver along the light direction to compensate
      translate lookup along the sun direction by the trapezoidal bias
  RETURN lookup
```

**Invariants** — the operator is composed **right to left from eye space**: the G-buffer
holds eye-space positions, so the inverse view comes first, then the cascade's light
transform, then the texture adjustment. A rebuild whose G-buffer stores world positions or
depth drops or replaces the first factor and nothing else.

The y row is negated because texture coordinates run downward and clip coordinates run
upward. A device whose texture origin is already at the top removes that negation, and
nothing else in the matrix changes — this is the single most common difference between
fillings of this seam.

**Unrecovered**: the sign applied to the depth bias is inconsistent across the three paths —
the legacy path negates the near bias, the filtered path negates the far bias, and the
cascade path takes whatever its caller passes. The source marks the first of these as a
work-around for inverted culling in the far region. Which convention is intended is not
determinable from this file; a rebuild should pick one and adjust the configured values.

## `build_cloud_lookup` — the moving cloud shadow

**Contract** — builds a second matrix that projects a world position into the cloud texture,
so the sun can be occluded by the sky's own cloud layer without any shadow map.

```text
FUNCTION build_cloud_lookup() -> matrix
  frame := a camera-like basis looking along the sun direction,
           with "up" taken from the wind's compass direction
  wind_in_frame := the wind direction expressed in that basis, normalized
  cloud_drift   := cloud_drift + 0.003 * elapsed_seconds
  RETURN inverse view, composed with `frame`,
         scaled by 0.002 in x and y,
         translated along wind_in_frame by cloud_drift
```

**Invariants** — the scale of 0.002 means one unit of cloud texture covers 500 world units,
so the cloud pattern repeats every 500 metres. The drift is along the **wind** direction
expressed in the sun's frame, so cloud shadows slide across the ground the way the clouds
overhead do, and the rate of 0.003 texture units per second is 1.5 world metres per second —
a light breeze, regardless of the environment's configured wind speed. That the configured
wind *speed* is read and then not used is visible in the source; only its *direction* reaches
the matrix.

## `accum_direct_cascade` — the three-cascade sun

**Contract** — as `accum_direct`, but for *near*, *middle* and *far* cascades, and each
cascade claims its pixels with geometry, depth and stencil rather than with a quad's depth.
Takes the cascade's transform and the previous cascade's transform.

The masking, shadow lookup and cloud lookup are identical. Everything below is not.

### The cascade's volume

```text
IF the legacy-cascade compatibility option is on
    emit a full-screen quad, cull nothing               # exactly the older behaviour
ELSE
    source := the previous cascade's transform when this is the FAR cascade,
              otherwise this cascade's own transform
    emit the 8 corners of CASCADE_BOX_CORNERS transformed by the inverse of `source`
    emit CASCADE_BOX_FACES as indices
    cull one face set                                   # the near faces are discarded
```

**Invariants** — this is the decision the whole file turns on. A cascade's shadow map covers
a box in world space; drawing that box's far faces and depth-testing against the scene
selects exactly the pixels inside it, in three dimensions, with no per-pixel range test in
the program. The older sun could only split the screen by a plane; this splits it by a
volume, which is what lets the cascades be fitted to the view frustum rather than to
distance alone.

**The far cascade draws the *previous* cascade's box, not its own.** Its own box is enormous
and would cover the screen. Instead it rasterizes the middle cascade's box and inverts the
depth comparison, so it lights everything *beyond* where the middle cascade stopped. The far
cascade is defined as the complement of the others, and that is how the complement is drawn.

### The depth comparison and the stencil hand-off

```text
# which pixels this cascade may light
IF sub_phase is NEAR or MIDDLE
    depth comparison := admit pixels nearer than the rasterized face
ELSE IF sun z-culling is disabled
    depth comparison := admit everything       # rely on stencil alone
ELSE
    depth comparison := admit pixels farther than the rasterized face

# what this cascade does to the stencil on its way out
IF sub_phase is FAR
    stencil write := nothing
ELSE
    stencil write := clear every bit except the lowest, on pixels that were lit
```

**Invariants** — **each cascade consumes the pixels it lit.** The mask wrote the light
marker into the stencil of every sunlit pixel; the near cascade lights the ones inside its
box and then zeroes their marker back down to the G-buffer's 1, which the *next* cascade's
"stencil at least the marker" test rejects. Cascade overlap is therefore resolved by the
stencil, once, with no blending and no seam — and it is why **the cascades must run
near-to-far**. Running them in any other order lights the near band with the far cascade's
coarse shadow map.

The far cascade does not clear, because there is nothing after it.

When sun z-culling is disabled the far cascade admits every pixel and relies purely on the
stencil hand-off. That is correct but slower — the whole screen is shaded and then rejected
per pixel — and it exists because the depth-based variant is sensitive to how tightly the
middle cascade was fitted.

### The screen texture-generation matrix

The cascade path additionally hands the program a matrix taking a clip position to a screen
texture coordinate, because a pixel produced by rasterizing a *box* does not know its own
screen position the way a full-screen quad's interpolated coordinate does. The quad path
does not need it. This is the one constant that exists only because the geometry changed.

The far cascade also receives the view direction projected into shadow space and
renormalized in the plane, which the program uses to fade the shadow out toward the edge of
the last cascade rather than ending it at a hard line.

## `accum_direct_f` — the filtered sun

**Contract** — the alternative path taken when the sun-filter option is on. Structurally
identical to `accum_direct`, with one substitution: the sun is accumulated into a **screen
scratch target** rather than into the lighting accumulator. The luminance sub-phase
delegates to `accum_direct_lum`. Clears the scratch on the near sub-phase.

**Invariants** — the point is that the sun's shadow term becomes an image before it is added
to anything, so it can be softened as an image. Doing that in the lighting program would
need as many shadow-map taps per pixel as the kernel is wide; doing it afterwards costs one
extra screen-sized target and one extra full-screen pass, independent of kernel width.

**Notes** — this path still carries a half-texel offset in its texture-adjustment matrix
that the other two dropped. Under a device convention where a texture coordinate addresses a
texel's centre the offset is wrong and shifts every shadow by half a shadow-map texel; under
the older convention it was required. Its survival here is an oversight, not a choice.

## `accum_direct_lum` — the softening resolve

**Contract** — the final sub-phase of the filtered path. Binds the lighting accumulator and
adds the sun scratch into it through a five-tap kernel, reading the centre and four diagonal
neighbours at 0.6 of a pixel.

**Invariants** — the offsets are carried in the vertex stream, as seven coordinate sets, for
the same reason the combine pass carries its antialias taps there: they are affine in screen
position. A kernel radius of 0.6 pixels is deliberately sub-pixel — it softens the shadow's
edge without visibly blurring it, which is the difference between a filtered shadow and an
out-of-focus one.

**Notes** — the material element this selects is named for luminance, and the sub-phase is
named for luminance, but neither measures anything. The name is a leftover from an earlier
arrangement in which this pass also produced the sun's contribution to auto-exposure; that
measurement now happens inside bloom.

## `accum_direct_blend` — closing the sun

**Contract** — ends the sun's accumulation. On devices that cannot blend into a
16-bit-float target, copies the blend twin into the real accumulator with one full-screen
draw; on devices that can, does nothing. Always advances the light marker.

**Invariants** — the marker advance is unconditional and is the reason this must be called
even on capable devices. The marker advances by two per light so that consecutive lights do
not need a stencil clear between them; the sun is a light like any other in that
bookkeeping.

## `accum_direct_volumetric` — light shafts

**Contract** — adds the sun's volumetric contribution into the separate volumetric
accumulator. Runs only on the near and far sub-phases, only when light shafts are enabled.
Uses the **geometry and vertex offset the caller already emitted** rather than emitting its
own.

```text
FUNCTION accumulate_light_shafts(cmd_list, sub_phase, offset, shadow_lookup, z_min, z_max)
  IF light shafts are not being rendered this frame THEN RETURN
  IF sub_phase is neither NEAR nor FAR THEN RETURN
  bind the volumetric accumulator ; colour writes on

  material := the light-shaft description, or its min/max variant when a
              min/max shadow pyramid was built this frame
  constants: the sun direction and colour in eye space, `shadow_lookup`,
             the screen texture-generation matrix, and (z_min, z_max)

  IF sub_phase = NEAR
      depth comparison := admit pixels farther than the geometry
  ELSE
      depth comparison := admit everything

  draw the caller's geometry
```

**Invariants** — a light shaft is the sun's contribution to the *air* between the camera and
a surface, so it is integrated along the view ray between `z_min` and `z_max` and the
program marches the shadow map along that ray. The **min/max variant** reads a pyramid that
records, per region of the shadow map, the nearest and farthest depths in it — which lets
the march skip whole ranges where the shadow map is uniform. That is the entire optimization
and it is what
[`r3_rendertarget_create_minmaxSM.cpp`](../xrRender_R2/r3_rendertarget_create_minmaxSM.cpp.md)
exists to build.

The depth comparison is inverted from the lighting pass's: lighting wants the surface, shafts
want the air in front of it.

**Notes** — reusing the caller's vertex offset and bound geometry is the file's worst
coupling. The shaft pass is only ever called at the tail of a sun pass, so the geometry it
wants — the cascade box, or the full-screen quad — happens to be bound already; but nothing
states the dependency and moving either call site silently draws the wrong primitives. A
rebuild should pass the geometry explicitly.

The far region's integration range differs between the two suns: the legacy sun integrates to
the configured far-split distance, the cascade sun to the size of its last cascade. These are
usually but not necessarily the same number.

## Notes

**The four degenerate triangles.** The cascade box's index table has room for sixteen
triangles and twelve are filled; the draw submits all sixteen, so four triangles of
zero-valued indices are rasterized to nothing every cascade of every frame. Harmless, and
obviously unintended — a box has twelve triangles.

**The multisample shape** described in
[`r4_rendertarget_phase_combine.cpp`](r4_rendertarget_phase_combine.cpp.md) recurs verbatim
at each of the seven draw sites in this file: one per-pixel draw on interior pixels, then
either one per-sample draw whose program loops the samples or one draw per sample with the
sample mask narrowed. Under multisampling the stencil predicates change from "at least the
marker" to "exactly the marker", because the high stencil bit is carrying the
edge/interior flag and a magnitude comparison would confuse the two.

**The depth-bounds hint and the shadow-map fetch-four hint** are both present as struck-out
blocks at every draw site. Both were vendor-specific fast paths on the previous device
generation; neither has an equivalent here. They are kept in the source as a record of what
the pass would like to tell the device if it could — that it only cares about a known depth
slab, and that it is about to take four shadow taps at once — and a rebuild targeting a
device that can express either should reinstate them.
