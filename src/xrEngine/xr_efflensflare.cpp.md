# src/xrEngine/xr_efflensflare.cpp

> The sun's lens flare — five rays tell it how much of the sun is occluded, the weather tells it which flare to wear, and the result is one blend factor plus a screen-space gradient.

**Needs** — [`xr_efflensflare.h`](xr_efflensflare.h.md) · [`Environment.h`](Environment.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`xr_object.h`](xr_object.h.md) · [`editor_helper.h`](editor_helper.h.md) · [`xrCDB/xr_collide_defs.h`](../xrCDB/xr_collide_defs.h.md) · [`xrCDB/Intersect.hpp`](../xrCDB/Intersect.hpp.md) · [`xrMaterialSystem/GameMtlLib.h`](../xrMaterialSystem/GameMtlLib.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`xr_efflensflare.h`](xr_efflensflare.h.md)
**Tier floor** — T2 with a T1 dependency: the occlusion test reads triangle indices and per-bone material indices straight out of the collision database's arrays.

## Purpose

A lens flare is trivial to draw and hard to *gate*. The interesting work is deciding, every
frame, how much of the sun the camera can actually see — through foliage, through a chain
link fence, behind a wall, half behind a wall — and turning that into one smoothly-varying
number. Everything else in this file serves that number.

Three sub-problems, each with a decision worth keeping:

1. **Partial occlusion.** One ray gives a binary answer and the flare pops. Five rays in a
   small cross give six discrete levels and the flare fades.
2. **Translucent occluders.** Foliage should dim the sun, not extinguish it, so the
   occlusion test multiplies a per-material transparency rather than stopping at the first
   hit.
3. **Cost.** A ray query against the level's collision database is not cheap and this runs
   every frame. Each ray remembers the triangle that blocked it and re-tests that triangle
   first.

## State

```text
RECORD FlareSprite                 # one element of a flare chain
  opacity  : real
  radius   : real
  position : real                  # along the axis from the sun through screen centre;
                                   # 0 is at the sun, 1 at the opposite point
  texture  : text
  shader   : text

RECORD SunSprite : FlareSprite
  ignore_color : bool              # draw at full brightness, not tinted by the sun colour

RECORD FlareDescription            # one authored "sun", selected by the weather
  section   : text
  has       : set of {source, flares, gradient}
  source    : SunSprite            # the disc itself
  flares    : list<FlareSprite>    # the chain of ghosts
  gradient  : FlareSprite          # the full-screen glow
  rise_rate : real                 # 1 / seconds to blend in
  fall_rate : real                 # 1 / seconds to blend out

RECORD LensFlare
  palette        : list<FlareDescription>
  current        : optional<FlareDescription>
  state          : {none, idle, hiding, showing}
  state_blend    : real in [0,1]   # cross-fade between two authored suns
  visibility     : real in [0,1]   # smoothed occlusion result
  gradient_value : real            # screen-position-attenuated gradient opacity
  light_colour   : colour          # the weather's sun colour
  sun_direction  : unit vector     # towards the sun
  basis          : camera right, up, forward, plus the flare axis and centre
  ray_cache      : list<CachedRay> # one per ray: origin, direction, length, hit triangle
  should_render  : bool
  last_frame     : int             # guards against being stepped twice in one frame
```

Invariants:

- The per-frame step is idempotent within a frame, guarded by the frame stamp. It is
  reachable from more than one place in the frame and must not run twice.
- `state_blend` is clamped to [0,1] and is the weight of `current`. When it reaches zero
  during a hide, `current` is *replaced* and the state flips to showing — so a cross-fade
  between two suns is actually a fade-out followed by a fade-in, never an overlap.

## Loading a description

**Contract** — built from one configuration section. Three independent parts, each gated by
its own boolean key: the sun disc, the ghost chain, the gradient. The ghost chain is
authored as four parallel comma-separated lists — radii, opacities, positions, textures —
all sharing one shader; the texture list's length decides how many ghosts there are.

**Notes** — the key naming differs between games: the oldest calls the disc `source` and
the two newer ones call it `sun`. The loader reads whichever exists, and when *both* exist
reads `source` first and lets `sun` override. That is explicitly for adapted configuration
files where a modder copied an old section forward and left the old keys in place. This is
the chapter's recurring shape: **accept every generation's spelling, prefer the newest**.

The blend rates are stored as reciprocals of the authored times, with a small epsilon added
to the time before inverting so that an authored zero means "instant" rather than a
division by zero.

## `OnFrame` — the per-frame step

**Contract** — takes the mixed weather frame and a time-scale factor. Reads the sun's
direction and colour, advances the cross-fade, decides whether the flare is on screen at
all, computes the occlusion, and derives the gradient's opacity. Runs on the frame thread;
performs up to five ray queries against the collision database. Early-returns with
rendering disabled when there is no level, no current description, a black sun, or a sun
behind the camera.

```text
FUNCTION on_frame(weather, time_factor)
  IF already stepped this frame OR no level THEN RETURN
  sun_direction = -weather.sun_dir          # sun_dir is where light goes; we want where it is
  light_colour  = weather.sun_color
  wanted        = weather.lens_flare

  advance_cross_fade(wanted, time_factor)
  IF no current description OR light_colour is black THEN
    should_render = false; RETURN

  build camera basis (right, up, forward) from the device's camera
  IF dot(sun_direction, camera_forward) <= 0.01 THEN     # sun is behind or at the edge
    should_render = false; RETURN
  should_render = true

  # Project the sun onto a plane three quarters of the way to the far plane.
  # Everything downstream works in that plane, so the flare chain is a straight
  # line in world space rather than a screen-space construction.
  depth       = far_plane * 0.75
  screen_mid  = camera_position + camera_forward * depth
  sun_point   = camera_position + sun_direction * (depth / dot(sun_direction, camera_forward))
  flare_axis  = sun_point - screen_mid

  visibility_target = occlusion_probe()
  move visibility towards visibility_target at the fall rate, capped by the frame delta
  clamp visibility to [0,1]

  IF the description has a gradient THEN
    gradient_value = gradient_opacity(sun_point)
```

**Invariants** — the cutoff on the forward dot product is 0.01, not zero. At exactly zero
the sun is on the plane of the camera and the projection divides by it; the small positive
floor keeps the projected point finite. Any small positive value works.

The projection plane at three quarters of the far distance is arbitrary in magnitude and
load-bearing in *kind*: the flare geometry lives in world space at a fixed depth, so it
scales with the projection like everything else and needs no special screen-space path.

## The occlusion probe

```text
FUNCTION occlusion_probe() -> real
  # five directions: the sun, and four neighbours one spread-unit away in the
  # camera's right and up axes
  spread = 0.02
  total  = 0
  FOR i IN 0..4
    d = sun_direction + right * spread * offsets[i].x + up * spread * offsets[i].y

    IF cached[i] recorded a hit for a similar (origin, direction, length) THEN
      contribution = 0                                  # still blocked, no query
    ELSE IF the ray still hits the cached triangle THEN
      contribution = 0                                  # cheap re-test, no query
    ELSE
      contribution = 1
      query the collision database along d, visiting every hit:
        FOR EACH hit
          IF the hit is a dynamic object THEN
            transparency = the material of the bone that was hit, or 0 if none
          ELSE
            transparency = the material of the static triangle that was hit
            IF transparency is zero THEN remember this triangle in cached[i]
          contribution = contribution * transparency
          STOP visiting once contribution falls below the threshold
    total = total + contribution
  RETURN total / 5
```

**Invariants and decisions:**

- The five offsets are the centre plus the four axis neighbours — a plus sign, not a square
  — so the probe costs five rays and samples the sun's extent in both axes.
- The spread is **0.02** and the source comments that it *should* come from the weather and
  does not. That is an acknowledged shortcut: the apparent size of the sun is fixed
  regardless of what the weather says the sun's radius is. A rebuild should take it from
  the description's source radius, which is what the dead code alongside it did.
- **Occlusion multiplies, it does not stop.** Each hit contributes its material's visual
  transparency factor. Two layers of foliage dim more than one. The traversal stops early
  once the product falls below a threshold, which is a cost bound, not a semantic.
- **The cache stores only fully-opaque static triangles.** A translucent hit is not worth
  caching because it does not determine the answer by itself. This is why the cache is
  effective: the common case is the sun fully behind a wall, frame after frame, and that
  case becomes one triangle test.
- A hit on a dynamic object whose skeleton yields no bone is treated as **fully opaque**.
  Fail-closed: an unknown occluder blocks.
- The ray length is a fixed 1000 world units regardless of the far plane.

The smoothing towards the probe result uses a *rate limit* rather than an exponential
filter — the value moves at most a fixed amount per second towards the target, and snaps
when it is within epsilon. A rate limit is the right choice here because the input is a
six-level staircase: an exponential filter would round the steps but still show them, while
a rate limit makes the transition linear and the staircase invisible.

## The cross-fade between suns

```text
none    -> adopt the weather's flare, go to showing
showing -> blend up at the description's rise rate; at 1, go to idle
idle    -> IF the weather now wants a different flare, go to hiding
hiding  -> blend down at the fall rate; at 0, adopt the new flare and go to showing
```

**Notes** — the blend rates come from the *outgoing* description in both directions, which
means a sun with a slow fall rate takes its time leaving regardless of what replaces it.
The time factor multiplied into both rates is the weather system's own time scale, so a
fast-forwarded day fades its flares proportionally faster.

## The gradient's opacity

**Contract** — the full-screen glow fades out as the sun approaches and passes the edge of
the screen, rather than vanishing at the boundary.

```text
FUNCTION gradient_opacity(sun_point) -> real
  p = project sun_point to normalised screen coordinates, y flipped
  inner = 0.5, outer = 2.5
  kx = 1; IF |p.x| > inner THEN kx = (outer - |p.x|) / (outer - inner)
  ky = 1; IF |p.y| > inner THEN ky = (outer - |p.y|) / (outer - inner)
  IF |p.x| > outer OR |p.y| > outer THEN RETURN 0
  RETURN kx * ky * state_blend * gradient.opacity * visibility
```

**Notes** — the attenuation begins at half the screen half-width (so, a quarter of the way
out from centre) and reaches zero at five times that, which is well off screen. The effect
is that the glow is at full strength only when the sun is near the middle of the view and
tapers over a long distance as the player turns away. The two numbers are pure feel and are
not derived from anything.

## `Render`

**Contract** — draws through the renderer's lens-flare filling, with three independent
switches for the disc, the ghost chain and the gradient. A console variable disables the
ghost chain alone, because that is the part players most often find objectionable; the disc
and gradient are part of the sky and stay.

## `AppendDef`

**Contract** — resolves a description by section name, building it once and caching it.
Same three-way configuration fallback as the lightning effect: the dedicated suns file, the
global configuration, then whichever exists.

## `OnDeviceCreate` / `OnDeviceDestroy`

**Contract** — builds and tears down every sprite's material across the whole palette. Both
are driven by the device-reset signal, so a graphics device loss rebuilds every flare
material. This is the concrete instance of the `DeviceReset` obligation described in
[`pure.h`](pure.h.md).

## Notes

The save side is declared, resolves the correct output path for both configuration layouts
— and then does nothing. Like the lightning effect's, it is unfinished.

The flare palette is loaded in full at startup in non-shipping builds so the weather editor
can enumerate it; shipping builds build descriptions on demand.
