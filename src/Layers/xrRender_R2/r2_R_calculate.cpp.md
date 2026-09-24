# src/Layers/xrRender_R2/r2_R_calculate.cpp

> The frame's preparation step: derive the screen-area thresholds every level-of-detail
> decision uses, find the camera's sector, gather the lights, and start the background
> visibility phases.

**Needs** — [`r2.h`](r2.h.md) ·
[`xrRender/r__dsgraph_structure.h`](../xrRender/r__dsgraph_structure.h.md) ·
[`xrRender/Light_DB.h`](../xrRender/Light_DB.h.md) ·
[`xrRender/HOM.h`](../xrRender/HOM.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: arithmetic and task dispatch; the device is not touched.

## Purpose

Everything that must be true before the frame graph starts. It runs on the simulation
side of the frame, ahead of drawing, and its last act is to launch the visibility walks
that [`r2_R_render.cpp`](r2_R_render.cpp.md) later joins.

The one piece of real arithmetic here is the derivation of the screen-area thresholds.
Every "is this object big enough to bother with" decision in the renderer — whether to
draw it at all, whether to sort it, which level of detail to pick, whether to use the
detailed material, how far parallax survives — is expressed as a *fraction of the screen*
in the configuration, and must be converted once per frame into the squared-distance
units the visibility walk compares against.

## State

```text
# Module-scope, recomputed every frame, read by the visibility walk and by material
# selection. All are squared-distance thresholds derived from a screen-area budget.
screen_budget      : real   # pixels the reference object would cover at this camera
discard_threshold  : real   # below this apparent size an object is not drawn at all
dont_sort_threshold: real   # below this, a sorted object may be drawn unsorted
lod_a, lod_b       : real   # the two level-of-detail switch points
glod_start, glod_end : real # the geometry-level-of-detail fade band
hzb_vs_tex_threshold : real # when to test against the software occluder vs. sample it
detail_distance    : real   # how far parallax and detail textures survive
```

**Invariants** — all of these are reciprocals of the screen budget, so they scale together
with resolution and field of view. The budget itself is the product of the target's pixel
count, a field-of-view correction, and the user's level-of-detail multiplier.

## `calculate`

**Contract** — called once per frame before rendering. Recomputes the thresholds, resolves
the two parallelism options, detects the camera's sector, updates the light database,
pulls in any light whose volume touches the camera even through a portal, joins the
software occluder's background rasterization, and starts the main, rain and sun phases.
Returns immediately if this is the first frame after a device reset.

```text
FUNCTION calculate()
  # --- thresholds -----------------------------------------------------------------
  fov_correction = (90 / field_of_view)^2
  screen_budget  = target_width * target_height * fov_correction * lod_multiplier
  FOR EACH configured screen-fraction s
      threshold = (s / 3)^2 / screen_budget      # note the divisor, below
  detail_distance = parallax_range * screen_budget / (1024 * 768)

  options.distortion       = options.distortion_enabled
  options.parallel_calculate = user setting
  options.parallel_render    = user setting AND backend records commands in parallel

  IF first frame after reset THEN RETURN

  # --- where is the camera ---------------------------------------------------------
  IF the camera moved since the last detection
      sector = the sector containing the camera position
      IF sector is valid
          IF sector changed THEN notify the game layer
          remember it

  # --- lights -----------------------------------------------------------------------
  update the light database                       # ages out, re-sorts, re-packages
  FOR EACH light source whose volume contains the camera position
      re-associate it with the sector its own reference point is in
      IF it has a sector, add it to this frame's package
  # this catches a light in the next room whose volume reaches through the portal

  AWAIT the software occluder's background rasterization

  # --- phases ------------------------------------------------------------------------
  initialise main, sun (or legacy sun), rain
  run main
  run rain
  run sun
```

**Notes** — the division by three before squaring is not a fudge: the configuration states
these thresholds as the *diameter* an object subtends in a normalized unit where the
reference is three units, so dividing by three converts to a fraction and squaring moves
into the squared-area domain the comparison happens in. The detail distance is divided by
a fixed 1024×768 because the shipped default was tuned at that resolution; the ratio makes
parallax fade at the same apparent size at any other.

Field of view enters as the square of its ratio to ninety degrees, which makes the budget
a genuine *area*: halving the field of view doubles each linear dimension of a distant
object and so quadruples its coverage.

The light gather is a second pass over the database because the normal pass associates a
light with the sector containing its *origin*. A lamp behind a doorway has its origin in
the far room and would be culled by the portal walk, even though its sphere spills into
the camera's room. Querying by the camera position and re-associating catches exactly that
case.

## `main_phase`

**Contract** — the phase (see [`r2.h`](r2.h.md)) that walks the scene graph for the camera
view. Always active. Its calculate step fills the immediate context's visibility set; its
render step is empty, because the main view's drawing is issued inline by the frame graph.

```text
FUNCTION main_phase.initialise()
  calculate_in_parallel = parallel option AND NOT legacy sun AND NOT depth prefill
  draw_in_parallel      = false          # always on the immediate context
  active                = true

FUNCTION main_phase.calculate()
  configure the immediate context:
      phase = normal
      priorities 0 and 1, and capture wallmarks
      IF the legacy sun is active, record the receiver boxes into the coarse structure
      occluders on; this is the main pass
      start sector = the camera's sector
      portal traversal: occluder-cull, screen-area-cull, and fade portals
      spatial traversal: ordered, renderables and light sources
      view position = the camera; transform = the camera's full transform
      frustum = the view frustum; precise portal clipping on
  build the visibility set
```

**Notes** — two things forbid running this walk in parallel. The legacy sun consumes the
*coarse receiver structure* the main walk records, and it cannot record into a shared
vector from another thread — the comment in the source calls it a show-stopper, and it is
the reason the legacy sun forces everything serial. The depth prefill shares the same
context and must finish first. The cascaded sun has neither problem, which is the main
practical argument for preferring it.

The walk requests renderables *and* light sources from the spatial database, because
visibility of a light's volume is decided by the same portal walk as the geometry's.
