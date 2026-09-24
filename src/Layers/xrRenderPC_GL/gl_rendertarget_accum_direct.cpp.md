# src/Layers/xrRenderPC_GL/gl_rendertarget_accum_direct.cpp

> Accumulating the sun: the stencil marking scheme, the cascade geometry, the depth predicate per cascade, and the three alternative paths through the same lighting pass.

**Needs** — [`gl_rendertarget.h`](gl_rendertarget.h.md) · [`gl_rendertarget_u_set_rt.cpp`](gl_rendertarget_u_set_rt.cpp.md) · [`r2_R_sun.cpp`](r2_R_sun.cpp.md) · [`../xrRenderGL/glR_Backend_Runtime.h`](../xrRenderGL/glR_Backend_Runtime.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`gl_rendertarget.h`](gl_rendertarget.h.md)
**Tier floor** — T1: it is the frame's heaviest shading pass, driven by per-pixel stencil arithmetic and a hand-built light volume.

## Purpose

The sun is the one light in the scene that reaches everything, so the deferred renderer gives it a pass of its own, run once per shadow cascade. This file is that pass.

Four decisions here are worth a rebuilder's full attention, and none of them is specific to this API:

**Masking before shading.** Before any cascade shades, one full-screen quad writes the current light's identifier into the stencil buffer for every pixel the sun could possibly light. Every subsequent draw tests against that identifier, so pixels the sun cannot reach — sky, previously-marked geometry — cost a stencil test rather than a lighting evaluation. The identifier is per-light and only reset when it wraps, which is why the stencil is almost never cleared.

**The light volume is a cuboid, not a quad.** After the mask, the cascade's shading draw is not a full-screen quad but a solid built from the inverse of the cascade's own light transform — the cascade's frustum, back in world space. Rasterizing that volume restricts shading to the pixels the cascade actually covers, which is what makes a three-cascade scheme cost roughly one full-screen pass in total rather than three.

**The depth predicate encodes the cascade order.** The near and middle cascades test greater-or-equal, so they shade only in front of what has already been written; the far cascade either tests always (when depth culling is off) or less (when on). That one line is the whole cascade-overlap policy, and getting it wrong shows up as either double-lit seams or an unlit band at a cascade boundary.

**There are three routes through the same job**, selected by configuration: the ordinary route, a *filtered* route that renders the sun into a separate surface so an edge-aware filter can smooth its shadow before it is blended into the accumulator, and a *luminance* route that computes only what the exposure pipeline needs. A rebuild must implement at least the first; the other two are quality options.

## State

The file owns two constant tables that define the light volume.

```text
# The eight corners of a cascade's frustum in clip space. The near face sits
# at 0.7 rather than at the near plane, because the volume only has to cover
# the depth range this cascade is responsible for.
corners[8]  : the unit cube's corners with the near face pulled to depth 0.7

# Sixteen triangles: the six faces of that solid, two triangles each, with a
# winding chosen so the solid's OUTSIDE faces the camera.
facetable[16][3]
```

**Invariants** — the depth values in the corner table are for a clip space whose depth range is zero to one. This API's default depth range is minus one to one; the engine's projection matrices produce the zero-to-one convention and the table matches them. A rebuild that uses the API's native convention must rewrite this table.

## `accum_direct(command_list, cascade)`

**Contract** — accumulates one cascade of sun lighting into the accumulator. Delegates to the filtered route when sun filtering is configured. Installs the accumulator as the target, performs the stencil mask on the first cascade only, then draws the cascade's light volume with the sun material. Reads the sun's colour, direction and shadow transform from the sun light record prepared by [`r2_R_sun.cpp`](r2_R_sun.cpp.md).

```text
FUNCTION accum_direct(cmd, cascade) -> ()
  bind the accumulator as the target
  IF sun filtering is configured THEN accum_direct_filtered(cmd, cascade); RETURN

  element := the sun material's element for this cascade,
             swapped for the min/max variant when the near cascade is using
             min/max shadow maps this frame

  sun     := the scene's sun light
  L_dir   := the sun's direction, transformed into eye space and normalized
  L_colour, L_specular := the sun's colour and its derived specular intensity

  # --- mask, once, on the near cascade
  IF cascade IS the near one THEN
      cull nothing
      fill four vertices with a full-screen quad
      set the mask material, and pass it the sun direction scaled by the
        negative square root of the sun's perceptual intensity -- the shader
        uses this to decide which pixels the sun can possibly reach
      set stencil: pass where the existing value is <= the current marker,
        read mask 0x01, write mask 0xff, replace on depth-pass
      draw the quad
      IF multi-sampling THEN
          # The high bit of the stencil marks a multi-sample edge pixel; the
          # per-pixel pass takes the non-edge pixels and a second pass with
          # the edge bit set takes the rest. This backend supports ONLY the
          # optimized form -- the per-sample loop the other backend runs
          # aborts here.
          run the same draw twice with the edge bit distinguishing them
          restore the ordinary stencil predicate

  # --- depth clip plane for this cascade
  clip_depth := the projection of a point one near-cascade-distance
                along the camera's forward axis

  IF stencil recompression is configured AND this is the near cascade THEN
      stencil_optimize(cmd)

  # --- shade
  bind the accumulator; cull back faces; enable all colour channels

  # The texel-adjust matrix maps clip space into shadow-map texture space:
  # a half-scale-and-offset on each axis, the depth axis additionally scaled
  # by the cascade's depth range and biased by the cascade's own bias.
  texel_adjust := [ 0.5      0    0                0
                    0        0.5  0                0
                    0        0    0.5 × depth_range 0
                    0.5      0.5  0.5 + bias        1 ]

  shadow_transform := texel_adjust × cascade.light_transform × inverse_view
  IF this is the far cascade AND hardware shadow maps are on THEN
      translate the shadow transform along the light direction by the
      trapezoidal-map bias -- the far cascade's texels are large enough that
      the ordinary depth bias is not sufficient

  cloud_shadow_transform := a slowly drifting projection along the sun
      direction, oriented by the current wind direction and scaled down by a
      fixed factor, so the cloud layer's shadow crawls across the world

  screen_texgen := the matrix that turns a screen position into a geometry-
      buffer coordinate (identity world, current view and projection)

  # --- build the light volume
  upload the face table to the index stream
  FOR EACH of the eight corners
      transform it by the INVERSE of this cascade's light transform
      (for the far cascade with depth culling, by the PREVIOUS cascade's,
       so the far volume starts where the middle one ended)
  upload the eight transformed corners to the vertex stream

  set the sun element and pass it: the screen texgen, the eye-space light
      direction, the light colour with specular in the fourth component,
      the shadow transform and the cloud shadow transform
  IF this is the far cascade THEN
      also pass the view direction projected into shadow space, normalized
      in its first two components -- the far cascade fades its shadow out
      along the view direction and needs this to know which way that is

  # --- the cascade-order predicate
  IF near or middle cascade THEN depth predicate := greater-or-equal
  ELSE IF depth culling is off THEN depth predicate := always
  ELSE depth predicate := less

  # The near and middle cascades ZERO the stencil bits they consume, so the
  # far cascade shades only what they did not. The far cascade keeps them.
  IF far cascade THEN stencil write mask := 0x00, pass op := keep
  ELSE                stencil write mask := 0xFE, pass op := zero

  set the stencil predicate against the light marker and draw the volume
  (eight vertices, sixteen triangles), repeating for the multi-sample edge
  pass when multi-sampling is on

  IF advanced post-processing AND sun shafts are on AND this is the far
     cascade THEN accum_direct_volumetric(cascade, the volume's offset,
                                          shadow_transform)
```

**Invariants** — the stencil write masks are the cascade hand-off. `0xFE` clears every bit but the low one, so a pixel shaded by the near cascade is removed from the far cascade's set; `0x00` on the far cascade leaves the buffer intact for the lights that follow. Reversing them double-lights every near-field pixel.

**Invariants** — the far cascade's light volume is built from the *previous* cascade's inverse transform when depth culling is enabled. That is not a typo: the far volume must begin at the middle cascade's far plane, and the previous transform is what knows where that is.

**Notes** — a large amount of the original is commented-out state for facilities this backend does not have: depth-bounds testing, a four-tap depth fetch requested through an abused sampler state, and a texel offset compensating for a half-pixel sampling convention that this API does not have. All three were live on Direct3D 9 hardware. A rebuild needs none of them, and should read their absence as confirmation that the half-pixel offsets scattered through the quad-building code are also vestigial.

## `accum_direct_cascade(command_list, cascade, transform, previous_transform, bias)`

**Contract** — the same pass with the cascade's transform, previous transform and depth bias supplied by the caller rather than read from the sun record. This is the form the newer multi-cascade sun path calls; `accum_direct` is the form the older two-cascade path calls.

**Invariants** — **its mask quad's vertices are in a different order from `accum_direct`'s, vertically mirrored.** That is the window-origin disagreement surfacing in the one place it is easiest to get wrong: a quad whose texture coordinates are assigned in the top-down order samples the geometry buffer upside down. The two functions being otherwise near-identical while differing in exactly this makes it look like an inconsistency; it is the fix, applied to the newer path only. **The older path's ordering is therefore suspect**, and a rebuild should settle on one convention at the quad builder rather than per call site.

**Notes** — beyond that and the parameterized transforms, this function duplicates `accum_direct` almost line for line. The duplication is an artifact of adding a cascade scheme without retiring the old one; a rebuild should have one implementation taking the transforms as arguments.

## `accum_direct_filtered(command_list, cascade)`

**Contract** — the alternative route used when sun filtering is configured. Renders the sun's contribution into a separate full-size surface rather than directly into the accumulator, so that an edge-aware filter can smooth the shadow's aliasing before it is added. Delegates to the luminance route for the luminance sub-phase. Otherwise structurally the same pass, against a different target.

**Notes** — the filtered route exists because the shadow maps of the era aliased badly at the near cascade. It costs a full-size surface and an extra full-screen pass, and it is off by default on this backend's configuration.

## `accum_direct_blend(command_list)`

**Contract** — folds the separately-rendered sun contribution into the accumulator, for devices that cannot blend floating-point targets. Skipped entirely when the device can.

```text
FUNCTION accum_direct_blend(cmd) -> ()
  IF the device supports floating-point blending THEN
      increment the light marker; RETURN

  bind the accumulator with the multi-sample depth
  fill four vertices with a quad given DIRECTLY IN CLIP SPACE -- corners at
      ±1 with texture coordinates 0 and 1 -- not in pixel coordinates
  draw with the accumulator-mask material, stencil-tested against the marker,
      repeating for the multi-sample edge pass
  increment the light marker
```

**Invariants** — the light marker is incremented here regardless of which branch ran, because it marks the end of the sun's contribution and the following lights depend on it having advanced.

**Notes** — this quad is given in clip space while the mask quads above are given in pixel coordinates with a half-pixel offset. Both reach the same place because the corresponding vertex programs differ; the clip-space form is the one that is actually correct on this API and the pixel-coordinate form is inherited. The commented-out pixel-coordinate version is preserved directly above the live one, which makes the intent legible.

## `accum_direct_lum(command_list)`

**Contract** — the luminance sub-phase: shades the sun with the luminance element of the sun material, whose output feeds the exposure pipeline rather than the accumulator. Builds a wide-format quad carrying, per corner, the screen coordinate, a jitter coordinate, and four neighbourhood offsets at a fixed fraction of a pixel — the filter kernel is baked into the vertex attributes rather than computed in the shader.

**Invariants** — the neighbourhood offsets are scaled by a smoothing factor over the surface dimensions, so the kernel's footprint in *pixels* is resolution-independent. The factor is a tuned value with no recoverable derivation.

**Notes** — baking a filter kernel into vertex attributes is the idiom of a pipeline where the pixel stage could not compute addresses cheaply. It is harmless and a rebuild may compute the offsets in the shader instead.

## `accum_direct_volumetric(cascade, vertex_offset, shadow_transform)`

**Contract** — draws the sun's volumetric shafts, reusing the light volume the caller has already uploaded — which is why it takes a vertex offset rather than building its own geometry. Runs only for the near and far cascades, only when sun shafts are enabled and the scene needs them, and into the volumetric accumulation surface rather than the main one.

```text
FUNCTION accum_direct_volumetric(cascade, offset, shadow_transform) -> ()
  IF shafts are not needed this frame THEN RETURN
  IF cascade is neither the near nor the far one THEN RETURN

  bind the volumetric accumulator; enable all colour channels
  element := the volumetric sun material, swapped for its min/max variant
             when min/max shadow maps are in use this frame
  IF min/max maps are in use OR the new cascade scheme is active THEN
      cull back faces

  pass the eye-space sun direction, the sun colour, the shadow transform,
       a texgen built from the world-view-projection with a HALF-PIXEL
       offset folded into its translation, and the cascade's near and far
       distances as the volume's integration range

  depth predicate := greater for the near cascade, always for the far one
  draw the volume: four vertices and two triangles under the old cascade
       scheme, eight and sixteen under the new one
```

**Invariants** — the integration range differs between the two cascade schemes: the old scheme's far cascade starts at the near cascade's far distance, the new one's starts at zero. That is the difference between shafts that are sampled only beyond the near field and shafts sampled across the whole view, and it is the visible difference between the two schemes.

**Invariants** — the primitive count differs with the scheme for the same reason the geometry does: the old scheme draws a quad, the new one draws the cuboid. Drawing the wrong count reads past the uploaded vertices.

**Notes** — this is the one place a half-pixel offset is still *live* rather than commented out, folded into the texgen's translation. Whether it is needed on this API or is a leftover that happens to be below the visible threshold is **not recoverable**; the commented-out alternative immediately beside it is the version without the offset.

**Notes** — the multi-sample branch of this function is commented out entirely, including the non-optimized per-sample loop. Volumetric shafts and multi-sampling therefore do not combine on this backend.
