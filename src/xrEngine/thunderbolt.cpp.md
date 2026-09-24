# src/xrEngine/thunderbolt.cpp

> The lightning effect — places a bolt somewhere on the horizon at weather-driven intervals, and pushes its flash back into the sky, sun and fog colours for the duration.

**Needs** — [`thunderbolt.h`](thunderbolt.h.md) · [`Environment.h`](Environment.h.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`IGame_Persistent.h`](IGame_Persistent.h.md) · [`LightAnimLibrary.h`](LightAnimLibrary.h.md) · [`Render.h`](Render.h.md) · [`editor_helper.h`](editor_helper.h.md) · [`xrCDB/xr_area.h`](../xrCDB/xr_area.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`thunderbolt.h`](thunderbolt.h.md)
**Tier floor** — T2: configuration, random placement, a ray query and colour arithmetic.

## Purpose

Lightning is not a particle effect and not a light: it is a *weather* event with a visual,
a sound, and — the part that makes it worth a file — a transient override of the scene's
global lighting. For the fraction of a second a bolt is alive, the sky colour, the sun
colour, the fog colour and, on the deferred renderers, the *sun direction itself* are
replaced by the bolt's contribution.

That last one is the design decision the whole file exists to support: the bolt does not
add a light, it *becomes* the sun. Placing a real light large enough to illuminate a level
from the horizon is unaffordable; repointing the existing directional light at the bolt is
free and looks right.

## State

```text
RECORD ThunderboltDescription            # one authored bolt variant
  name          : text                   # its configuration section
  visual        : renderer model handle  # the lightning geometry
  sound         : audio handle           # optional
  colour_anim   : light animation handle # drives the flash colour over the bolt's life
  gradient_top    : Flare
  gradient_centre : Flare

RECORD Flare                             # one of the two glow sprites on a bolt
  opacity : real
  radius  : (real, real)
  shader  : text
  texture : text

RECORD ThunderboltCollection             # a named palette a weather frame selects from
  section : text
  palette : list<ThunderboltDescription>

RECORD ThunderboltEffect
  collections     : list<ThunderboltCollection>
  current         : optional<ThunderboltDescription>
  state           : {idle, working}
  transform       : 4x4                  # placement and scale of the live bolt
  direction       : unit vector          # from the bolt towards the camera; becomes the sun
  centre, size    : the bolt's midpoint and length
  phase           : real in [0,1]        # how far the strike has propagated visually
  life_time       : real                 # this bolt's duration, randomised per strike
  current_time    : real
  next_strike_at  : real                 # global clock
  enabled         : bool                 # mirrors "the current weather has thunderbolts"
```

Authored parameters, loaded once:

```text
altitude_range     : (real, real)   # radians; elevation above the horizon
delta_longitude    : real           # radians; spread about the anti-sun bearing
min_dist_factor    : real           # fraction of the far plane, capped at 0.95
tilt               : real           # radians; how far the bolt may lean from vertical
second_probability : real in [0,1]  # chance the next strike follows immediately
sky_color          : real           # how strongly the flash tints each of the three
sun_color          : real
fog_color          : real
```

Invariants:

- A strike is alive exactly while `current_time <= life_time`; outside that window the
  effect contributes nothing and is not drawn.
- `direction` points *from the strike towards the origin* by the time it is published,
  because it is consumed as a sun direction and the sun's direction is the direction light
  travels. It is built pointing outwards and inverted at the end of placement.
- The sun direction it publishes must point downwards. A bolt placed at or below the
  horizon would produce light from underground; the engine asserts rather than clamps,
  because the altitude range that allows it is an authoring error.

## Configuration, and three places it may live

The bolt palettes are read from two dedicated configuration files under the environment
directory, and the tuning parameters from a third. Each has a fallback into the global
configuration under a differently-named section. This exists because the three supported
games organise their weather data differently and all three must load unmodified, which is
the chapter-wide rule.

The lookup order for a palette by name is: the dedicated collections file if it has the
section; otherwise the global configuration; otherwise whichever of the two exists. A
rebuild needs the *fallback chain*, not the file names, and should note that the section
names differ between the two locations (`environment` versus `thunderbolt_common`) — the
data, not just its home, is game-specific.

In non-shipping builds every collection in the file is loaded at startup rather than on
demand, so the weather editor can list them. That is a tooling concession with a real
memory cost and is correctly compiled out.

## `Bolt` — placing a strike

**Contract** — chooses a variant, a position, an orientation and a duration; finds the
strike's ground end with a ray query; schedules the next strike; and plays the sound.
Called only from the frame step, only when idle, and only when the current weather has a
palette.

```text
FUNCTION place_bolt(weather)
  state = working
  base_life = weather.bolt_duration
  life_time = base_life +/- 50% of itself      # no two strikes last the same time
  current_time = 0
  variant = random member of weather.palette

  # Bearing: opposite the sun, plus a spread. Lightning reads as coming from the
  # weather front, and the sun marks where the front is not.
  bearing   = sun_bearing + half_turn +/- delta_longitude
  elevation = random in altitude_range
  distance  = random in [far_plane * min_dist_factor, far_plane * 0.95]

  direction = unit vector at (bearing, elevation)
  origin    = camera_position + direction * distance

  # Lean: a small random tilt about two axes, plus a free spin about the third
  lean = (random in +/-tilt, random full turn, random in +/-tilt)
  down = lean applied to straight down

  size = far_plane * 2
  ray query from origin along down, limited to size   -> shortens size to the hit
  centre = origin + down * size / 2
  transform = translate(origin) * rotate(lean) * scale(size)

  IF random < second_probability THEN
    next_strike_at = now + life_time            # a double strike: no sound, no gap
  ELSE
    next_strike_at = now + bolt_period +/- 30% of it
    play variant.sound at origin

  direction = -direction                        # now points sunwards, ready to be the sun
```

**Invariants** — the maximum distance factor is capped at 0.95 of the far plane and the
minimum is clamped never to exceed that cap. A bolt at or past the far plane is clipped
away entirely and the effect silently does nothing, which is the bug the cap prevents.

**Notes** — the *second strike* mechanism is why a bolt sometimes flashes twice in quick
succession: with the configured probability the next strike is scheduled to begin the
instant this one ends, and crucially **plays no sound**. Real lightning's second stroke
arrives inside the first thunderclap.

The sound's attenuation distances are set from the strike distance — audible from half the
distance to twice it — and its playback delay is the distance divided by 300. That is the
speed of sound in metres per second, and it is why distant lightning is seen before it is
heard. This is the file's one piece of physics and it is worth keeping.

The ray query falls back to an artificial ground plane at height zero when it hits nothing,
so a bolt over open water or off the edge of the collision geometry still terminates
somewhere rather than extending for twice the far plane.

## `OnFrame`

**Contract** — called once per frame with the *mixed* weather frame (see
[`Environment.cpp`](Environment.cpp.md)), which it reads and writes. Starts strikes, ages
the live one, and applies the flash to the weather frame's colours. Mutating its input is
the point: this effect is a stage in the weather pipeline, not a consumer of it.

```text
FUNCTION on_frame(weather)
  IF (weather has a palette) != enabled THEN     # the weather just changed
    enabled = weather has a palette
    next_strike_at = now + bolt_period +/- 50% of it   # re-randomise, don't fire at once
  ELSE IF enabled AND now > next_strike_at AND state is idle THEN
    place_bolt(weather)

  IF state is working THEN
    IF current_time > life_time THEN state = idle
    current_time = current_time + frame_delta

    flash = variant.colour_anim sampled at (current_time / life_time)
    phase = clamp(1.5 * current_time / life_time, 0, 1)

    weather.sky_color = clamp(weather.sky_color + flash * sky_color_weight, 0, 1)
    weather.sun_color = weather.sun_color + flash * sun_color_weight
    weather.fog_color = weather.fog_color + flash * fog_color_weight
    IF the renderer is deferred THEN
      weather.sun_dir = direction
```

**Invariants** — the state transition to idle happens *before* the time advance, so the
last frame of a strike still renders. A strike therefore lasts one frame longer than its
nominal life, which is both harmless and deliberate: checking after the advance would drop
the final frame of the colour animation, which is where the fade to nothing lives.

**Notes** — the visual phase runs at **1.5 times** the life fraction and is then clamped,
so the bolt is fully drawn by two-thirds of its life and the remaining third is pure fade.
That is what makes a strike read as "flash, then glow" rather than as a uniform ramp.

Only the sky colour is clamped after the flash is added; the sun and fog are not. They feed
a tone-mapped path that tolerates over-range values, and clamping them would flatten the
flash's peak.

The sun override is applied only on the deferred renderers, because the older forward path
has no single directional light to repoint.

## `AppendDef`

**Contract** — resolves a palette by section name, building it on first request and
returning the cached one afterwards. An empty name yields nothing. Called by the weather
loader as it reads each weather frame's palette reference.

## `Render`

**Contract** — draws the live bolt through the renderer's thunderbolt filling; does nothing
when idle. The filling is a plug provided by each graphics backend and is granted access to
this object's internals, which a rebuild should replace with an explicit render description
handed across the boundary.

## Editor surface

**Contract** — debug-build panels that expose the tuning parameters and each variant's
flare settings live. Angles are shown in degrees and stored in radians, and changing a
flare's shader or texture name tears down and rebuilds its material immediately.

**Notes** — the save side is declared and **empty in every one of its four
implementations**. The editor can change these values in memory but cannot write them back;
whatever was intended here was never finished. A rebuild should either implement it or drop
the editor.
