# src/Layers/xrRender_R2/render_phase_sun.cpp

> The cascaded sun: three shadow maps of fixed world size, each fitted to a chained slice
> of the view frustum, snapped to texels so shadows do not crawl, and accumulated
> near-to-far.

**Needs** — [`r2.h`](r2.h.md) · [`r2_types.h`](r2_types.h.md) ·
[`r2_R_sun_support.h`](r2_R_sun_support.h.md) ·
[`r3_rendertarget_phase_smap_D.cpp`](r3_rendertarget_phase_smap_D.cpp.md) ·
[`r3_rendertarget_create_minmaxSM.cpp`](r3_rendertarget_create_minmaxSM.cpp.md) ·
[`xrRender/r_sun_cascades.h`](../xrRender/r_sun_cascades.h.md) ·
[`xrRender/light.h`](../xrRender/light.h.md) ·
[`xrCore/Threading/ParallelFor.hpp`](../../xrCore/Threading/ParallelFor.hpp.md)
**Used by** — [`r4_rendertarget_accum_direct.cpp`](../xrRenderPC_R4/r4_rendertarget_accum_direct.cpp.md)
**Tier floor** — T1: it builds projection matrices against a specific clip-space depth
convention and snaps to device texel centres.

## Purpose

The sun is a directional light: it has no position and its shadow frustum is unbounded, so
the only question is *which* region of the world gets shadow-map resolution. The answer is
three nested regions — cascades — of fixed world size, each covering the slice of the view
frustum the previous one stopped at.

Two things make this implementation distinctive. The cascades are *chained*: each one
starts where the previous one ended, by carrying forward the frustum rays the previous
fit clipped. And the projection is snapped to a texel grid every frame, without which the
shadow's edge samples a different texel each frame as the camera moves and the shadow
boundary visibly crawls.

## State

```text
RECORD SunPhase                      # one of the phases described in r2.h
  cascades       : three Cascade
  sun            : the sun light, borrowed from the light database
  needs_shafts   : bool              # decided each frame from the weather
  saved_chain_mode : bool            # the far cascade's chain flag, saved and restored
  contexts       : one command context per cascade

RECORD Cascade
  transform   : matrix      # the fitted view-projection
  rays        : four rays   # the view-frustum edges, advanced to where this fit ended
  size        : real        # the world-space side of the region, in metres
  bias        : real        # depth bias, proportional to size
  reset_chain : bool        # start from the camera again rather than from the previous
```

**Invariants** — cascade sizes are 20, 40 and 160 metres and the bias of each is its size
multiplied by a single small negative constant. Tying bias to size is what makes one
constant serve all three: a larger cascade covers more world per texel and needs a
proportionally larger bias to clear the same slope. The first cascade always resets the
chain; the others inherit unless told otherwise.

## `initialise`

**Contract** — sets the cascade sizes and biases, takes the sun from the light database,
and decides whether the phase runs at all. When it runs, pre-allocates one command context
per cascade and picks up the parallelism options.

```text
FUNCTION initialise()
  cascade sizes = 20, 40, 160 ; bias of each = size * (-0.0000025)
  cascade 0 resets the chain
  sun = the light database's sun
  active = the sun is enabled in settings AND its colour's gloss-derived intensity > 0
  IF the sun is baked into lightmaps THEN active = false
  IF NOT active THEN RETURN
  allocate one command context per cascade
  calculate_in_parallel = the parallel-calculate option
  draw_in_parallel      = the parallel-render option
```

**Notes** — the activity test runs the sun's colour through the same curve that derives a
light's specular scale. A sun whose colour is black contributes nothing and is skipped
entirely, which is what happens at night. The size progression 20 / 40 / 160 is not
geometric — a commented-out loop would have grown each by a factor of four — and the
middle cascade is closer to the near one than the ratios suggest. This is hand tuning
against the shipped levels' scale and is worth keeping as given.

## `calculate`

**Contract** — fits all three cascades and walks the scene graph for each. Runs on a
worker thread when parallel calculation is enabled, fanning the three walks out further.
Reads the camera transform and the sun direction; writes each cascade's transform and each
context's visibility set.

```text
FUNCTION calculate()
  needs_shafts = the weather wants light shafts this frame
  IF needs_shafts
      save the far cascade's chain flag and force it to reset
      # shafts march through the far cascade and need it anchored at the camera,
      # not chained behind the middle one

  # --- a stable light basis ---------------------------------------------------
  direction = the sun's direction, normalized
  right = the x axis, or the z axis if that is nearly parallel to the direction
  up    = direction x right, renormalized; right = up x direction
  light_view = a camera looking along `direction` from the sun's nominal position
  distance   = how far the camera is below the plane through the sun's position

  FOR EACH cascade, near to far
      # --- the region this cascade must cover ---------------------------------
      centre_of_projection = camera position, pushed 1200 m back along the sun
      IF this is the first cascade, or it resets the chain
          rays = the four view-frustum edges, from the near-plane corners toward
                 the far-plane corners, in world space
      ELSE
          rays = the rays the previous cascade left behind

      projection = an orthographic box of `size` across, from 0.1 m to
                   distance + sqrt(2) * size deep
      cuboid = the unit cube pulled back through (projection * light_view),
               with only its four side polygons
      fit the cuboid against the rays (see r2_R_sun_support.h), obtaining
          the culling planes and a shift in the light's plane
      leave the advanced rays for the next cascade

      # --- rebuild the view with the shift and snap to texels -----------------
      light_view = a camera at (sun position + shift) looking along the direction
      transform  = projection * light_view
      frustum    = the culling planes
      snap the transform (below)
      record the transform; the shadow rectangle is the whole map

  FOR EACH cascade, possibly in parallel
      configure its context: phase = shadow, priority 0 (and 1 with translucent
          shadows), start sector = the level's largest, transform, frustum, and
          view position = the centre of projection
      build the visibility set
```

### Texel snapping

**Contract** — adjusts the cascade transform by a sub-texel translation so that the
projected world aligns to a fixed grid in shadow-map space. Pure arithmetic on the
transform.

```text
FUNCTION snap(transform, shift)
  # quantize the camera position in world space first, on a 4 m grid
  aim = each component of the camera position floored to a 4 m cell, at the cell centre
  aim_in_texels = aim through the transform, then through the viewport matrix
  shift_in_texels = the light-plane shift, as a direction, through the same
  # the shift's sign, at a 4-texel granularity
  shift_in_texels = (+4 or -4 per axis by sign, 0 in depth)
  # the fractional part of the aim, in 4-texel units, scaled back to texels
  residual = frac(aim_in_texels / 4) * 4, in x and y only
  residual = residual - shift_in_texels
  back-transform the residual into world space and translate the transform by its
      negation
```

**Invariants** — the grid is four texels, not one, and the world pre-quantization is four
metres. Both must hold: snapping to single texels is not enough when the *shift* from the
cuboid fit moves by a fraction of a texel between frames, and quantizing the aim point in
world space first is what keeps the snap stable when the camera moves smoothly rather than
jumping between grid cells.

**Notes** — this is the most fragile arithmetic in the chapter and the least
self-explanatory. What it is defending against is shadow crawl: the shadow map is a
discrete sampling of the world, and if the sample grid moves relative to the world between
frames, every shadow edge shimmers. Snapping the projection so the grid is world-locked is
the standard cure; the complication here is that the cuboid fit also translates the light
every frame, and that translation must be folded into the snap or it reintroduces the
crawl it was supposed to remove. The four-texel granularity and the four-metre
pre-quantization are the empirical sizes that made it stable, and a sign-test variable
left in the source as a tunable constant suggests they were found by experiment.

## `render`

**Contract** — renders each cascade's shadow map. When cascades cannot share an array
texture, each one is also accumulated immediately, because the next cascade will overwrite
the map. Runs the cascades in parallel when parallel drawing is enabled.

```text
FUNCTION render()
  IF NOT active THEN RETURN
  IF needs_shafts THEN restore the far cascade's saved chain flag

  FOR EACH cascade, possibly in parallel
      context = this cascade's
      IF its priority-0 or priority-1 sets are non-empty
          bind this cascade's slice; clear it
          world = view = identity; projection = this cascade's transform
          draw priority 0
          IF the sun-details option is on, draw the detail layer
          IF there is translucent work
              rebind for the coloured-mask pass; draw priority 1; draw the sorted set
      IF the device has no target arrays THEN accumulate this cascade now
```

**Notes** — setting view to identity and putting the whole fitted transform in the
projection slot is not a shortcut; the fit produced one combined matrix and splitting it
back into a view and a projection would be arbitrary. The constant-binding machinery only
needs the product.

## `flush`

**Contract** — when the device has target arrays, accumulates all three cascades now that
all three slices exist. Then invalidates the immediate command list and restores the
camera transforms, because the cascade passes left the light's transforms bound.

## `accumulate_cascade`

**Contract** — adds one cascade's contribution to the accumulator, selecting the program
variant by the cascade's position in the chain, and building the min/max pyramid first for
the near cascade when light shafts want it. Releases the cascade's context afterwards.

```text
FUNCTION accumulate_cascade(index)
  IF index is the near cascade AND the min/max pyramid is wanted this frame
      build the min/max shadow map from this cascade
  select this cascade's slice for reading
  IF index == 0     accumulate as NEAR,   with this cascade's transform twice
  ELSE IF index is not the last
                    accumulate as MIDDLE, with this transform and the previous one
  ELSE              accumulate as FAR,    with this transform and the previous one
  submit the context and release it
```

**Invariants** — every cascade but the first is given *two* transforms: its own and its
predecessor's. The program uses the predecessor's to test whether the pixel already fell
inside the previous, higher-resolution cascade, and discards it if so. That is how the
cascades compose without a blend and without visible seams — each one shades only the
band between itself and the one before. The near cascade is handed its own transform twice
so the test degenerates to "always shade", and the program needs no special case.

The accumulation itself lives in the backend (`accum_direct_cascade`), because the sun's
lighting program differs between the two; what this chapter fixes is which variant, with
which matrices, in which order.
