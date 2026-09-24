# src/xrEngine/Environment_misc.cpp

> What a weather frame contains, how two of them blend, where the sun is, and how the whole weather set is read from and written back to configuration.

**Needs** — [`Environment.h`](Environment.h.md) · [`Environment.cpp`](Environment.cpp.md) · [`IGame_Level.h`](IGame_Level.h.md) · [`xr_efflensflare.h`](xr_efflensflare.h.md) · [`thunderbolt.h`](thunderbolt.h.md) · [`Rain.h`](Rain.h.md) · [`Common/LevelGameDef.h`](../Common/LevelGameDef.h.md) · [`Render.h`](Render.h.md) · [Configuration format](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`Environment.cpp`](Environment.cpp.md) · [`Environment_editor.cpp`](Environment_editor.cpp.md)
**Tier floor** — T2: configuration parsing, interpolation and an astronomical model. The chunked binary read of the modifier file is the only byte-level part.

## Purpose

Three separable jobs share this file because they share the frame record: reading a frame from configuration, blending two frames, and the sun-position model. The fourth job here — writing the whole weather set back out — exists for the in-game weather editor and is the only *writer* of shipped-format configuration anywhere in the engine.

The dominant complication throughout is that three games shipped with three different spellings of the same data, and all three must load. Wherever a key has two or three names, that is what is happening; the recipe names them rather than hiding them, because a rebuild that loads only the newest spelling loads only one of the three games.

## `EnvModifier.load`

**Contract** — Reads one local override volume from the level's modifier file: centre, radius, strength, and the six overridable quantities. A version word read from the file's first chunk decides whether a per-field enable mask follows; in older files there is none and every field is enabled.

```text
RECORD EnvModifier
  position     : vector3
  radius       : real
  power        : real
  far_plane    : real
  fog_color    : vector3
  fog_density  : real
  ambient      : vector3
  sky_color    : vector3
  hemi_color   : vector3
  enabled      : set of {view_dist, fog_color, fog_density, ambient, sky, hemi}
```

**Notes** — The enable mask arrives at file version 0x0016 and the default for an older file is *all enabled*. That default is load-bearing: the fields are all present in older files and were all meant to apply, and treating the absent mask as empty would make every older level's local weather overrides vanish.

**Notes** — The modifier file is a chunked container whose chunk identifiers are consecutive integers starting at zero, with chunk zero carrying the version word only if it is exactly one word long. A chunk zero of any other size is a modifier. That overloading is an artifact of adding a version to a format that had none.

## `EnvModifier.absorb`

**Contract** — Accumulates one modifier's contribution into this one, attenuated by distance from a viewpoint, and reports the attenuated strength. Contributes nothing outside the radius. Enabled fields propagate their enable bit into the accumulator, so the blender can tell which quantities any modifier touched at all.

```text
FUNCTION absorb(m, viewpoint) -> real
  d2 = squared distance from viewpoint to m.position
  IF d2 >= m.radius^2 THEN RETURN 0
  attenuation = 1 - sqrt(d2) / m.radius          # linear, 1 at the centre, 0 at the rim
  strength    = m.power * attenuation
  FOR EACH field enabled in m
    accumulate m.field * strength into this.field, and set this field's enable bit
  RETURN strength
```

**Notes** — Attenuation is linear in distance, not squared and not smoothstepped. A modifier therefore has a visible hard edge in its *derivative* at the rim. This is what the shipped levels were authored against.

**Notes** — Contributions from overlapping modifiers add without bound; the blender divides by one plus the total strength afterwards, which is what keeps the result finite. The pair only makes sense together.

## `EnvFrame.load`

**Contract** — Reads one authored time-of-day frame from a configuration section. The section's *name* is the timestamp and is parsed as `HH:MM:SS` with each field range-checked; a malformed name is fatal. Every other key is read by name, with a second spelling accepted for the older game generation, and a handful of optional keys defaulted. Resolves the frame's lens-flare, thunderbolt and ambient references into shared definitions, creating them on first use. Validates that every colour is within a plausible range and warns — does not fail — when one is not.

Key points where the data is not what it looks like:

- **Cloud colour** is five numbers, not four: red, green, blue, alpha and a *multiplier*. The three colour channels are multiplied by half the multiplier and the alpha is preserved. Half, because the authored values are in a range where 2 is full brightness — see the range check below.
- **Sky colour** in the older generation is likewise halved on load; in the newer it is not. The two generations authored the same visual in different units.
- **Colour range check** — every colour channel is expected to lie in 0..2, and a value outside that logs a warning naming the section. Two, not one: these are high-dynamic-range values and the renderer expects the headroom.
- **Sun direction** is either read directly as a two-angle pair, which pins it and disables the astronomical model for this frame, or read as an altitude/longitude pair which the model will override.
- **Sun azimuth** falls back to an engine-wide setting when the frame does not name one, and is clamped to a full turn.
- **Cloud rotation** defaults to the sky rotation rather than to zero, so a frame that rotates the sky rotates the clouds with it unless it says otherwise.
- **Tree sway amplitude** accepts two further spellings from two well-known total conversions. The engine is a drop-in replacement for modded installations as well as retail ones.
- **Thunderbolt period and duration** are read only when the frame actually names a thunderbolt collection, since they are meaningless without one.

**Notes** — Two texture names are derived from one: the sky texture, and the same name with a suffix marking the small, low-resolution variant used for the environment cube map. The suffix is a naming convention in the shipped texture set, not a format.

## `EnvFrame.save`

**Contract** — Writes a frame back into a configuration section, in the spelling of whichever generation this cycle came from, skipping frames marked as synthetic padding. Angles are written in degrees although they are held in radians, because that is what the shipped files contain. The sun is written back in whichever of its two forms the frame was loaded in, so a round trip does not silently convert a pinned sun into a computed one.

## `EnvFrameMixer.blend`

**Contract** — Produces the current environment from two frames, a blend weight, the accumulated modifier and its total strength. Almost every field is a straight linear interpolation; the exceptions are what matters.

```text
FUNCTION blend(A, B, f, modifier, total_power)
  fi = 1 - f
  attenuation = 1 / (total_power + 1)        # the environment's own share

  # Every modifier-affected field follows the same shape: interpolate, and only
  # if some modifier enabled that field, add the modifier and re-normalize.
  far_plane = (fi*A.far + f*B.far) * visibility_setting
  IF modifier enabled view_dist THEN far_plane = (fi*A.far + f*B.far + modifier.far) * visibility_setting * attenuation
  ... likewise fog colour, fog density, ambient, sky colour, hemisphere colour

  # Fog planes are derived, not authored:
  fog_near = (1 - fog_density) * 0.85 * fog_distance
  fog_far  = 0.99 * fog_distance

  # Discrete choices, not blends:
  lens_flare  = IF f < 0.5 THEN A.flare ELSE B.flare
  thunderbolt = IF f < 0.5 THEN A.bolts ELSE B.bolts
  ambient_set = randomly A or B, with probability (1-f) of A

  # Sun: either the astronomical model, or a blend of two authored directions
  IF the renderer wants a moving sun AND dynamic sun is enabled THEN
    (sun_dir, sun_blend) = sun_direction(current_time, azimuth)
    sun_color = sun_color * sun_blend
  ELSE
    sun_dir = normalize(lerp(A.sun_dir, B.sun_dir, f))

  # The packed environment colour the renderer reads, doubled with a small bias:
  source = IF older_generation THEN sky_color ELSE hemi_color
  env_color = (source * 2 + epsilon, blend weight in the fourth channel)
```

**Invariants** — The sun must point downward (its vertical component negative) after blending; an upward sun means an authored frame or the model has produced a sun below the horizon and every shadow in the frame is wrong.

**Notes** — Modifier application is *conditional per field* and the condition is "did any modifier in range enable this field", not "is there a modifier in range". The commented-out unconditional versions are still in the source and are the older behaviour: they applied the attenuation to every field whether or not a modifier touched it, which dimmed the whole world near any modifier. The conditional form is the fix.

**Notes** — Fog near is 85% of fog distance scaled down by density, and fog far is 99% of it. The two constants shape the fog ramp: the 0.99 keeps the far fog plane just inside the far clip so geometry never pops out of an unfogged gap, and the 0.85 with the density term makes denser fog start closer. Neither is authored anywhere.

**Notes** — The ambient set is chosen *randomly* between the two frames, weighted by the blend. That is not an error: ambient sets are discrete collections of sounds and particle effects that cannot be blended, and a random pick weighted by the transition produces a crossfade in aggregate — near the start of the transition you mostly hear the old set, near the end mostly the new. Choosing deterministically at the halfway point, as the lens flare and thunderbolt do, would make the switch a single audible cut. The consequence is that the ambient set changes every frame during a transition, and the ambient scheduler must tolerate that.

**Notes** — The doubling and epsilon bias on the environment colour match a shader convention: the renderer expects the value in a range where 0.5 is neutral, and the epsilon prevents an exactly-zero channel, which the shader divides by.

**Notes** — Which colour feeds the environment colour differs by game generation. The older generation used the sky colour and the newer the hemisphere colour, and the flag travels with the weather cycle rather than with the build, because one installation can contain cycles of both kinds.

**Notes** — Blended texture names are produced by *concatenating* both names with a separator, which the renderer then splits to bind two textures and cross-fade between them. A name pair, not a name. If either side is empty the nearer single name is used.

## `sun_direction`

**Contract** — Given the time of day and an azimuth offset, computes the sun's direction and a brightness factor that fades the sun out near the horizon. A real solar-position model, evaluated for a fixed latitude and longitude. Returns a downward-pointing direction.

```text
FUNCTION sun_direction(time_of_day, azimuth_offset) -> (direction, brightness)
  # Day angle. The fractional-day term makes the declination drift slowly
  # across successive in-game days rather than repeating exactly.
  g = radians((360 / 365.25) * (180 + time_of_day / DAY_LENGTH))

  declination      = third-order Fourier series in g     # degrees
  time_correction  = second-order Fourier series in g    # minutes, the equation of time

  longitude = -30.4 degrees        # the Chernobyl exclusion zone
  latitude  =  50.27 degrees

  hour_angle = (time_of_day / 3600 - 12) * 15 + longitude + time_correction
  wrap hour_angle into (-180, 180]

  cos_zenith = sin(lat)*sin(dec) + cos(lat)*cos(dec)*cos(hour_angle), clamped to [-1,1]
  elevation  = 90 degrees - acos(cos_zenith)
  compute the compass azimuth from the same triangle, add azimuth_offset
  IF hour_angle < 0 THEN azimuth = full_turn - azimuth      # morning is the mirror branch

  # The sun is never allowed exactly onto the horizon, and fades out over 1..3 degrees.
  elevation = max(elevation, 1 degree)
  brightness = clamp((elevation - 1 degree) / (3 degrees - 1 degree), 0, 1)
  direction = from (azimuth, -elevation)
  RETURN (direction, brightness)
```

**Notes** — The latitude and longitude are the game's real setting, the Chernobyl exclusion zone in northern Ukraine. They are the reason the sun tracks a plausible arc rather than a straight overhead sweep, and a rebuild that changes them changes every shadow in the game.

**Notes** — The two Fourier series are a standard closed-form approximation of solar declination and the equation of time. Their coefficients are not derivable from anything else in the repository and are not the project's own; they are the published series. A rebuild may substitute any solar-position model of comparable accuracy.

**Notes** — The day angle uses time-of-day divided by a day *added to a half-turn*, which means the model is evaluated at a notional day-of-year that advances by a fraction of a degree per in-game day. The game has no calendar, so the declination is effectively fixed near its value a half-year from the epoch; the term exists because the formula it was taken from has one.

**Notes** — Clamping the elevation to one degree rather than allowing the sun below the horizon is what prevents a sun-direction of zero length at dawn and dusk and the divide-by-zero that follows. The one-to-three-degree fade is a presentation choice: it dims the sun out before the clamp becomes visible.

## `append_ambient`

**Contract** — Returns the shared ambient set with a given name, loading it on first request. Sets are shared because many frames reference the same one and each carries live sound handles.

## `EnvAmbient.load` / `sound_channel.load` / `create_effect`

**Contract** — Read an ambient set: a list of sound channels and a list of timed particle effects, with their scheduling bounds. This is where three generations of configuration diverge most.

- **Sound channels** may be named by three different keys. In the oldest generation there is no separate channel section at all — the ambient section *is* the channel — and the loader flags that case and reads the channel's fields out of the ambient's own section.
- **Distance range** is either one two-component value or two separate keys, and an inverted pair is swapped rather than rejected.
- **Period** is four numbers: the bounds for the delay before the *first* play, and the bounds for the delay between subsequent plays. It arrives as a four-component value (middle generation), as a two-component value duplicated into four (oldest generation), or as four separate keys (newest). The older forms are in seconds and are converted to milliseconds; the newest is already in milliseconds. Both pairs must be ordered low-to-high or load fails.
- **Effects** carry a lifetime in seconds converted to milliseconds, a particle effect name, a spatial offset, a gust factor, an optional sound, and an optional wind blast with a strength, a compass direction and fade-in and fade-out times. An effect without a wind blast gets a zeroed blast pointing forward.

**Invariants** — An ambient set must have at least one sound channel or at least one effect; an empty set is a data error and fails the load.

**Notes** — A zero-width period range is the idiom for "never": the schedulers that read these report a delay of zero only when the bounds are ordered, and treat an unordered pair as "do not schedule". That is why the accessors test the ordering rather than assuming it.

## `load_modifiers` / `unload_modifiers`

**Contract** — Read the current level's modifier file if it exists — an absent file is normal — then reconcile every ambient set against this level's own ambient overrides. Unloading simply empties the list.

## `load_level_specific_ambients`

**Contract** — A level may ship a file that redefines ambient sets by the same names the global set uses. For every ambient already loaded, the most specific definition wins: the level's file, then the global ambient file, then the general configuration. An ambient whose winning source differs from the one it was loaded from is destroyed and reloaded from the new source.

**Notes** — The comparison is by *source file name*, not by content, so an ambient already loaded from the right file is left alone and its live sound handles survive a level change. That is the only reason the reconciliation is written this way rather than as an unconditional reload.

## `load_weathers` / `load_weather_effects`

**Contract** — Build the cycle and effect tables. Two layouts are read and merged: the newer one, where every file in a weather directory is a cycle and every section in it is a frame; and the older one, where a single configuration section lists cycles by name, each naming a section that lists frames by timestamp. Cycles from the older layout are tagged so the blender uses the older environment-colour convention. Both are idempotent — a non-empty table returns immediately. Fails if no cycle exists at all, or if any cycle has fewer than two frames. Finishes by activating the first cycle in name order.

**Notes** — Effect lists are padded with a synthetic frame at each end, one stamped at midnight and one at the end of the day, both marked not-to-be-saved. They exist so the splice in [`Environment.cpp`](Environment.cpp.md) always has four slots to overwrite and so an effect that does not cover the whole day still interpolates at its edges.

**Notes** — The initial cycle is whichever sorts first by name, which is arbitrary. It is immediately replaced by the level's own weather when a level loads, so it only shows in the brief window before that.

## `load` / `unload`

**Contract** — Create (or destroy) the three weather visual effects — rain, lens flare, thunderbolt — and load (or release) the cycle and effect tables. Unloading also clears the renderer's environment resources and invalidates the frame pair.

## `save` / `save_weathers` / `save_weather_effects`

**Contract** — Write the whole weather set back to configuration, in the layout each cycle was loaded from. For the older layout this additionally regenerates the index sections and the include list that tie the files together, and rewrites each frame's section name by substituting underscores for the colons in its timestamp, because a colon is not legal in a section name in that layout.

**Notes** — This is the weather editor's save path and the only place the engine writes shipped-format configuration. A rebuild that does not ship the editor drops it, but should keep the observation that the two layouts are not interchangeable: a cycle authored in one cannot be written in the other without also converting the colour units.
