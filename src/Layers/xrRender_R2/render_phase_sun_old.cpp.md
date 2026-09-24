# src/Layers/xrRender_R2/render_phase_sun_old.cpp

> The legacy sun: two shadow maps — a snapped near region and a trapezoid-warped far
> region refitted around the frame's actual casters and receivers.

**Needs** — [`r2.h`](r2.h.md) · [`r2_types.h`](r2_types.h.md) ·
[`r2_R_sun_support.h`](r2_R_sun_support.h.md) ·
[`r3_rendertarget_phase_smap_D.cpp`](r3_rendertarget_phase_smap_D.cpp.md) ·
[`r2_rendertarget_phase_accumulator.cpp`](r2_rendertarget_phase_accumulator.cpp.md) ·
[`xrRender/light.h`](../xrRender/light.h.md) ·
[`xrRender/r__sector.h`](../xrRender/r__sector.h.md)
**Used by** — [`r2_R_sun.cpp`](../xrRenderPC_GL/r2_R_sun.cpp.md) · [`r4_rendertarget_accum_direct.cpp`](../xrRenderPC_R4/r4_rendertarget_accum_direct.cpp.md)
**Tier floor** — T1: projective matrix construction in a specific clip-space convention,
against device-sized viewports.

## Purpose

The sun the first two games shipped with. Instead of nested cascades of fixed world size,
it draws two shadow maps: a *near* one covering the first few metres of the view frustum,
snapped to texels; and a *far* one whose projection is warped so that shadow-map
resolution is distributed by distance rather than uniformly — trapezoidal shadow mapping —
and then refitted around the bounding boxes of the frame's actual shadow casters and
shadow receivers.

It is selected when the installed game data's sun program expects it (see
[`r2.cpp`](r2.cpp.md)). It cannot run its visibility walk in parallel, because it consumes
a structure the main view's walk records, and that serialisation is the strongest practical
argument for the cascaded sun.

## State

The same phase record as [`render_phase_sun.cpp`](render_phase_sun.cpp.md), plus:

```text
  casters : list<box>     # coarse bounding boxes recorded while walking for shadows
  # and it reads the renderer's `coarse_structure`, the receivers the main walk recorded
```

**Invariants** — the caster list is filled by the shadow walk and the receiver list by the
main walk; both must be cleared after the far map is fitted, or they accumulate across
frames. Only one command context is used, for both regions in sequence.

## `initialise`

**Contract** — as the cascaded sun's, with two differences: parallel calculation is forced
off, and one context is allocated rather than three. The cascade table is still filled with
the same three sizes and biases even though only two regions are drawn, because the light's
shadow-transform array is indexed by the same sub-phase enumeration.

## `render`

**Contract** — draws the near region, then the far region, then the optional filtered
resolve. Each region walks the scene graph, renders its shadow map, and accumulates
immediately — the two regions share one map, so the second overwrites the first.

```text
FUNCTION render()
  IF NOT active THEN RETURN
  render_near()
  render_far()
  render_filtered()
```

**Notes** — near before far is required, not stylistic: both write the same shadow map, and
the accumulation of each must happen before the next overwrites it. The cascaded sun's
array target is what removed this constraint.

## `render_near`

**Contract** — fits, renders and accumulates the near shadow region: the first few metres
of the view frustum, from the camera's near plane out to a configured near distance.

```text
FUNCTION render_near()
  frustum = the camera's projection narrowed to [near plane, configured sun-near]
  hull    = that frustum's eight corners, pulled back to world space, with its six faces
  planes  = extend the hull to infinity along the sun direction   # the caster volume
  centre_of_projection = camera position, 1200 m back along the sun

  light_view = a camera along the sun direction with a stable perpendicular basis
  box        = the hull's bounding box in light space, grown slightly
  projection = an orthographic box over it, with the near plane pushed back 1000 m
               so casters above the hull still render
  transform  = projection * light_view

  snap: project the camera position, round it to whole texels through the viewport
        matrix, and translate the transform by the difference
  scissor: project every hull point through (viewport * transform) and take the
           bounding rectangle, clamped to the map — this is the region of the map
           the near region actually occupies

  walk the scene graph from the level's largest sector with that transform and the
      culling planes, occluders OFF
  render the shadow map; accumulate as NEAR (building the min/max pyramid first when
      light shafts want it); restore the camera transforms
```

**Invariants** — occluders are disabled for this walk. A caster hidden from the camera can
still cast into view, so culling against the camera's occlusion buffer would delete
shadows.

**Notes** — the scissor rectangle is computed and stored on the light but the near pass
does not set a viewport from it; the accumulation reads it to build its lookup matrix. The
near region therefore *may* occupy less than the whole map, and the lookup must know how
much.

The near-plane pushback of a thousand metres is the same constant the far region uses and
is simply "far enough that no caster in any shipped level is outside it".

## `render_far`

**Contract** — fits, renders and accumulates the far shadow region, covering the frustum
out to the weather's far plane. Three optional stages shape its projection.

```text
FUNCTION render_far()
  frustum = the camera's projection out to min(configured sun-far, weather far plane)
  hull    = its corners and faces in world space
  planes  = the hull extended to infinity along the sun     # the caster volume
  a plain orthographic fit over the hull gives the fallback transform

  walk the scene graph from the level's largest sector, recording every caster's
      coarse bounding box
  IF the ignore-portals option is on
      additionally add every sector's root visual unconditionally
      # a level whose portals are mis-authored otherwise loses distant shadows

  # --- stage 1: trapezoidal warp, unless the sun is nearly along the view --------
  IF |cos(angle between view and sun)| < 0.99 AND the warp is enabled
      build a light-space basis with the light direction as "up" and the view as
          the forward axis
      rotate the view frustum's eight extrema into it
      translate along depth so every caster is in front of the near plane, using the
          casters' depth bounds unioned with the frustum's
      fit an orthographic box to the result
      find the centre line of the projected frustum (near-plane centre to far-plane
          centre), rotate it onto the horizontal axis, and scale both axes to unit
      choose a projection point beyond the near end of that line at a distance
          derived from the frustum's extent and a tuning parameter, placed so that
          the 80-percent line of the frustum maps to the middle of the map
      shear so the trapezoid is symmetric about the centre line, then apply a
          two-dimensional homogeneous projection from that point and rescale to
          clip space
      transform = view * light_basis * ortho * trapezoid

  # --- stage 2: focus refit, if enabled ------------------------------------------
  IF the focus option is on
      casters_box   = the bounding box of every recorded caster, in the warped space
      receivers_box = the bounding box of the receivers the main walk recorded,
                      clipped against the camera frustum, in the warped space
      widen receivers_box with the corners of a short guaranteed-range frustum
          # so that the first 20 m always has shadow, whatever the refit says
      clamp: receivers may only shrink the box in x and y, casters only in depth
      transform = transform * (the box, mapped to the unit cube)

  render the shadow map; accumulate as FAR (min/max pyramid first if wanted)
```

**Invariants** — the refit may only *shrink* the receiver box, never grow it past the unit
cube, and may only shrink the caster depth range into [0, 1]. The recorded caster boxes are
coarse — they are whole visuals' bounds, not triangles — so letting them expand the fit
would waste resolution on empty space. The guaranteed range is a hard floor: a refit that
excluded the first twenty metres would remove the shadows the player actually sees.

**Notes** — the trapezoidal warp is disabled when the sun is nearly parallel to the view
direction, because the centre line degenerates and the construction divides by its length.
The 0.99 threshold is a cosine, so roughly eight degrees.

The 80-percent line — the tuning constant is a negative fraction used to place the
projection point — is where the warp puts the most resolution. It is a distance along the
frustum, not a fraction of the screen, and the intent is that the region a player looks at
most (mid-distance, not the feet and not the horizon) gets the densest sampling.

This whole construction is a faithful implementation of published trapezoidal shadow
mapping, with the focus refit bolted on. A rebuild that chooses the cascaded sun instead
must still keep this one, because levels shipped expecting it; but nothing else in the
engine depends on its internals.

## `render_filtered`

**Contract** — when the sun-filter option is on, binds the accumulator and runs one more
sun accumulation in the luminance sub-phase. Off by default; it is a command-line switch.

**Notes** — what this extra pass does lives in the backend's sun accumulation. Its
existence is the decision recorded here: the sun may be resolved twice, once normally and
once through a filtered variant, which softened the shadow boundary on hardware without
depth-comparison filtering.

## `flush`

**Contract** — submits and releases the single context, then invalidates the immediate
command list so the next pass re-binds everything.
