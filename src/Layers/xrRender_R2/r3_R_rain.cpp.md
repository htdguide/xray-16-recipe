# src/Layers/xrRender_R2/r3_R_rain.cpp

> The rain light: a directional light aimed straight down whose shadow map is never used
> to shadow anything — it records which surfaces the sky can see, so they can be made wet.

**Needs** — [`r2.h`](r2.h.md) · [`r2_types.h`](r2_types.h.md) ·
[`r2_R_sun_support.h`](r2_R_sun_support.h.md) ·
[`r3_rendertarget_phase_smap_D.cpp`](r3_rendertarget_phase_smap_D.cpp.md) ·
[`r3_rendertarget_draw_rain.cpp`](r3_rendertarget_draw_rain.cpp.md) ·
[`xrRender/light.h`](../xrRender/light.h.md) ·
[`xrRender/r__dsgraph_structure.h`](../xrRender/r__dsgraph_structure.h.md) ·
[`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) ·
[Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it builds an orthographic projection in a specific clip-space
convention and snaps it to device texel centres.

## Purpose

Wet surfaces need one fact per pixel that nothing else in the G-buffer carries: *is there
open sky above this point?* A puddle forms on the pavement and not under the awning, and
the difference is occlusion along the vertical, which no light in the scene measures.

So the renderer invents a light. It has no colour, contributes nothing to the accumulator
and is never seen. It is a directional light pointing straight down, it renders exactly
one shadow map, and that map is the sky-visibility oracle the wetness rewrite in
[`r3_rendertarget_draw_rain.cpp`](r3_rendertarget_draw_rain.cpp.md) samples. Reusing the
shadow machinery for a visibility query is the whole idea of this file; nothing else about
it is unusual.

It is a phase like the sun's (see [`r2.h`](r2.h.md)), so its scene walk runs off the frame
and is joined in the frame graph immediately before the sun.

## State

```text
RECORD RainPhase                  # one of the phases described in r2.h
  light        : Light            # the invented light: direction, position, one
                                  #   shadow transform and one map rectangle
  rain_factor  : real             # the weather's current rain density, 0..1
  context      : one command context
  active       : bool
```

**Invariants** — the light's direction is exactly straight down every frame; it is never
derived from the weather or the sun. Its position is the camera's, lifted by a large fixed
offset, which only matters because the light basis is built from a position and a
direction; nothing depends on the distance. The shadow rectangle is always the whole map —
unlike a spot light, the rain light does not share an atlas.

## `initialise`

**Contract** — decides whether the phase runs, and allocates its command context if so.
Reads the weather's rain density and the camera's motion.

```text
FUNCTION initialise()
  rain_factor = the weather's current rain density
  active = the wet-surface option is on
       AND rain_factor > 0
       AND the camera moved or turned since the last frame
  IF NOT active THEN RETURN
  pick up the parallel calculate/render options
  allocate one command context
```

**Invariants** — **a stationary camera skips the phase entirely.** The saved camera
position and direction are the ones the engine recorded at the start of the previous
frame; when neither has changed the rain map from that frame is still valid, because the
map is fitted to the camera and to nothing else. This is the only phase in the renderer
with a *reuse* condition, and it is what makes wetness affordable.

**Notes** — the consequence a rebuilder must not miss is that the rain shadow map
**survives frames**. The frame graph still runs the wetness rewrite every frame (see
`flush` below); only the map's *rendering* is skipped. A rebuild that clears the map per
frame, or that allocates it from a transient pool, will flicker wetness whenever the
player stands still.

## `calculate`

**Contract** — fits the rain light's orthographic volume to the camera, then walks the
scene graph into the phase's context. Reads the camera transform and the configured
wet-surface far distance; writes the light's shadow transform and the context's visibility
set. Runs on a worker thread when parallel calculation is enabled.

```text
FUNCTION calculate()
  direction = straight down
  position  = the camera's position, raised by a large fixed offset

  # --- the region wetness may cover -------------------------------------------
  far = the configured wet-surface far distance
  view = the camera's projection narrowed to [near plane, far], times the camera view
  hull = the eight corners of that frustum pulled back to world space, with its
         six faces
  planes = extend the hull to infinity along `direction`      # the caster volume
  # radius of the sphere that contains that narrowed frustum, from the pyramid
  # relation  radius = edge_length_squared / (2 * height)
  radius = far * (1 + tan(half_fov)^2 + tan(half_fov_across)^2) / 2
  # `half_fov_across` is the engine's own horizontal field of view: the vertical
  # angle in degrees multiplied by the aspect ratio, then halved. That is not the
  # trigonometrically correct horizontal angle, but it is the convention every
  # projection in the renderer is built with, so it must be used here too.

  centre_of_projection = camera position, pushed 1200 m back along the direction,
      then slid horizontally by `radius` along the camera's forward axis

  # --- the projection ----------------------------------------------------------
  light_view = a camera at `position` looking along `direction`, with a stable
               perpendicular basis (see the sun; the degenerate axis is handled
               the same way)
  box = the hull's bounding box in light space, grown slightly
  # the horizontal extent is DISCARDED and replaced by the bounding sphere,
  # recentred ahead of the camera by the same horizontal slide
  box.x = [-radius, +radius] shifted by radius * camera_forward.x
  box.y = [-radius, +radius] shifted by radius * camera_forward.z
  projection = an orthographic box over it, whose depth range starts 1000 m before
               the box's nearest depth and ends 2000 m past it
  transform = projection * light_view

  snap: project the world origin through the transform and the viewport matrix,
        floor it to whole texels, and translate the transform by the difference
  the map rectangle is the whole map

  configure the context: phase = shadow, priority 0 only, start sector = the
      level's largest, transform, culling planes, view position = the centre of
      projection
  build the visibility set
```

**Invariants** — the depth extent of the orthographic box is *not* fitted: it is a fixed
3000-metre slab hung around the box's nearest depth (a thousand metres before it, two
thousand past). Fitting it would be tighter, but a caster above the visible hull — a roof
over a courtyard the camera is standing in — must still be in the map, and the slab is
simply "deep enough for any shipped level".

**Invariants** — the horizontal fit is a *square from the bounding sphere*, not the
hull's own box, and it is slid forward by the sphere's radius along the camera's
horizontal heading. The two must agree: the same slide appears in the centre of
projection and in the box, and changing one without the other decentres the map. Fitting
the hull instead would give a long thin trapezoid whose texel density varies wildly with
view angle, and wetness is a boolean per pixel — a tight fit buys nothing, while a square
one keeps the map's texel size constant as the camera turns.

**Invariants** — the snap quantises the *world origin*, not the camera, and to whole
texels rather than the sun's four. The map is fixed to a world grid so that a wet patch
does not shimmer as the camera moves; the origin serves because the projection's
orientation never changes (the light always points down) so a single world point is enough
to pin the grid.

**Notes** — priority 0 only, and translucent shadows off: wetness is a yes/no question
and a pane of glass neither blocks rain nor is made wet by it.

**Notes** — the 1200-metre pushback for the centre of projection is the same constant the
sun uses, and means the same thing: the visibility walk needs a *point* to start from, far
enough outside the world that everything is in front of it.

## `render`

**Contract** — renders the rain shadow map from the phase's visibility set, into the
context's own command list. Skipped when the walk found nothing. Writes depth only.

```text
FUNCTION render()
  IF NOT active THEN RETURN
  IF the visibility set is empty THEN RETURN
  bind the rain shadow map as the depth target and clear it
  world = view = identity; projection = the fitted transform
  draw priority 0
```

**Notes** — world and view identity with the whole fitted transform in the projection slot
is the same convention the sun uses, and for the same reason: the fit produced one matrix
and splitting it would be arbitrary.

## `flush`

**Contract** — submits and releases the phase's context, then — **whether or not the phase
was active** — runs the wetness rewrite on the immediate context. Restores the camera
transforms first, because the shadow pass left the light's in place. Records the frame the
light was used in.

```text
FUNCTION flush()
  IF active
      submit and release the context
  IF the wet-surface option is off THEN RETURN
  IF rain_factor is zero THEN RETURN          # no rain, nothing to rewrite

  invalidate the immediate command list
  restore world = identity, view and projection = the camera's
  bind albedo as the colour target with the multisample depth
  draw_rain(the rain light)                   # r3_rendertarget_draw_rain.cpp
```

**Invariants** — the two early exits here are *not* the same test as `active` in
`initialise`. `active` additionally requires that the camera moved; the rewrite must run
on every frame it rains regardless, against whatever map exists. Collapsing the two tests
into one is the natural-looking simplification that breaks the reuse optimisation.

**Notes** — the target bound before the rewrite is albedo, but the rewrite rebinds
everything it writes itself. The bind here only guarantees a sane pass dimension for the
first thing the rewrite does.
