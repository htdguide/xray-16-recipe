# src/editors/xrWeatherEngine/editor_environment_weathers_time.cpp

> One keyframe: the frozen set of fields the engine interpolates, the grid page that authors them, and the blend that reports what the engine is actually using.

**Needs** — [`editor_environment_weathers_time.hpp`](editor_environment_weathers_time.hpp.md) · [`editor_environment_weathers_weather.hpp`](editor_environment_weathers_weather.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`editor_environment_ambients_manager.hpp`](editor_environment_ambients_manager.hpp.md) · [`editor_environment_suns_manager.hpp`](editor_environment_suns_manager.hpp.md) · [`editor_environment_thunderbolts_manager.hpp`](editor_environment_thunderbolts_manager.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`xrEngine/Environment.h`](../../xrEngine/Environment.h.md) · [`xrEngine/xr_efflensflare.h`](../../xrEngine/xr_efflensflare.h.md) · [`xrEngine/thunderbolt.h`](../../xrEngine/thunderbolt.h.md) · [`ide.hpp`](ide.hpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`editor_environment_weathers_time.hpp`](editor_environment_weathers_time.hpp.md)
**Tier floor** — T1: writing a texture name re-creates a device resource in the middle of a frame.

## Purpose

**The file that defines what a weather keyframe is.** Everything else in this module exists
to get an author to this grid page and to get its fields safely onto disk. The keyframe
layout written here must match what the engine reads, because the engine reads what this
tool writes.

## State

See [`editor_environment_weathers_time.hpp`](editor_environment_weathers_time.hpp.md).

## The keyframe record on disk

```text
RECORD KeyframeSection                       # one configuration section per keyframe
  section name              : text     # HH:MM:SS — the time this keyframe takes effect

  ambient                   : text     # a record name in the ambients file
  ambient_color             : vec3     # colour
  sun                       : text     # a record name in the suns file
  sun_color                 : vec3     # colour
  sun_altitude              : real     # degrees on disk, radians in memory
  sun_longitude             : real     # degrees on disk, radians in memory
  sun_shafts_intensity      : real     # 0..1
  sky_texture               : text     # texture path, extension dropped
  sky_color                 : vec3     # colour
  sky_rotation              : real     # degrees on disk, radians in memory
  hemisphere_color          : vec4     # colour plus a fourth component
  clouds_texture            : text     # texture path, extension dropped
  clouds_color              : vec4     # colour; the fourth component is transparency
  clouds_rotation           : real     # degrees on disk, radians in memory
  fog_color                 : vec3     # colour
  fog_density               : real     # 0..1
  fog_distance              : real     # invariant: less than far_plane
  far_plane                 : real
  water_intensity           : real     # 0..1
  rain_color                : vec3     # colour
  rain_density              : real     # 0..1
  thunderbolt_collection    : text     # a record name in the collections file
  thunderbolt_duration      : real     # seconds
  thunderbolt_period        : real     # seconds
  wind_direction            : real     # degrees on disk, radians in memory
  wind_velocity             : real     # metres per second, 0..1000
```

**Invariants** — every angle is stored in degrees and held in radians; fog distance should
be less than the far plane (stated in the field's own description, not enforced); the
identifier and the section name are the same text.

**Notes** — **This layout is frozen.** The game reads these sections; a rebuilt editor that
renames a field, changes an angle's unit, or drops one breaks every installed cycle file.

The asymmetry between reading and writing is the single most surprising thing in this
module and it is deliberate: **writing emits all twenty-six fields; reading reads only
three of them here.** Everything else is read by the engine's own keyframe loader, which
this file calls through after taking its three. The three it keeps — the ambient, sun and
thunderbolt-collection *names* — are kept because the base loader resolves them to
references and discards the text, and the editor needs the text to show in a picker and to
write back.

The original carries the full read sequence commented out, field by field, beside the call
that replaced it. That is the history of the decision: the editor once duplicated the
loader and now delegates. A rebuild should delegate from the start and keep only the three
names.

Note also the field renamed in transit: the in-memory blend duration is written as
`thunderbolt_duration`/`thunderbolt_period`, and the fourth hemisphere component is
written under the name `hemisphere_color` while the sky's is `sky_color` — the names on
disk are the frozen ones, whatever the memory layout calls them.

## `load`, `load_from`, `save`

```text
FUNCTION load(config)
  ambient                = config.text(identifier, "ambient")
  sun                    = config.text(identifier, "sun")
  thunderbolt_collection = config.text(identifier, "thunderbolt_collection")
  base_keyframe.load(environment, config)          # reads everything else

FUNCTION load_from(section : text, config, new_id : text)
  identifier = section        # borrow the source section's name so load() finds it
  load(config)
  identifier = new_id         # then put the intended name back

FUNCTION save(config)
  write all twenty-six fields INTO section named identifier
  angles converted radians -> degrees on the way out
```

**Contract** — `load` reads the section named by this keyframe's own identifier.
`load_from` exists for paste and reload, where the fields come from a section with a
*different* name than the keyframe keeps; it borrows the name, reads, and restores.

**Notes** — The borrow-and-restore is a consequence of the loader addressing configuration
by the object's own identifier rather than by an argument. A rebuild passes the section
name as a parameter and `load_from` disappears.

## `fill` — the authoring surface

**Contract** — builds the keyframe's grid page: roughly thirty rows in nine named groups —
`properties`, `sun`, `hemisphere`, `clouds`, `ambient`, `fog`, `rain`, `thunderbolts`,
`wind`. Every row binds to the live keyframe, so editing a cell changes the running
weather on the next frame.

```text
GROUP properties   : id (text, filtered through the cycle's naming rule)
GROUP sun          : colour; shafts intensity 0..1; altitude and longitude -360..360 degrees;
                     sun record, chosen from the suns list
GROUP hemisphere   : sky texture (file browser, .dds, extension dropped);
                     sky colour; hemisphere colour; sky rotation -360..360 degrees
GROUP clouds       : clouds rotation -360..360; clouds texture (file browser);
                     clouds colour; transparency 0..1
GROUP ambient      : ambient colour; ambient record, chosen from the ambients list
GROUP fog          : fog colour; far plane; fog distance; fog density 0..1;
                     water intensity 0..1
GROUP rain         : rain colour; rain density 0..1
GROUP thunderbolts : collection, chosen from the collections list; duration; period
GROUP wind         : direction -360..360 degrees; velocity 0..1000 metres per second
```

**Notes** — Three patterns in this page are worth lifting.

**Bounded rows are documentation.** `0..1` on a density, `-360..360` on an angle,
`0..1000` on a wind velocity: these are the only place the valid range of a weather field
is written down anywhere in the project. A rebuild that drops the bounds loses the spec.

**The three name rows pull their options from other sub-managers at paint time**, not from
a captured list — so a sun renamed in the suns page immediately offers its new name here.
The same pull-don't-mirror rule as the timeline.

**Rows that must react bind to accessors; rows that need not bind to the field.** Angles
bind to accessors because degrees must be converted to radians. Textures bind to accessors
because changing one has to re-create a device resource. Colours and densities bind
straight to the field, because writing them is all there is to do. The split is not
stylistic — it is where the side effects are.

## The three name setters and their side effects

```text
FUNCTION set_ambient(value : text)
  IF value == ambient THEN RETURN
  ambient = value
  resolved_ambient = environment.ambient_named(value)

FUNCTION set_sun(value : text)
  IF value == sun THEN RETURN
  sun = value
  lens_flare_id = lens_flare_library.append(environment, suns_config, value)

FUNCTION set_thunderbolt_collection(value : text)
  IF value == thunderbolt_collection THEN RETURN
  thunderbolt_collection = value
  thunderbolt_id = thunderbolt_library.append(environment, collections_config,
                                              thunderbolts_config, value)
```

**Contract** — each writes the name *and* re-resolves the reference the renderer uses, so
the change is visible in the same frame. Each returns early when the name is unchanged,
which matters: re-resolving allocates a library entry.

## The two texture setters

```text
FUNCTION set_sky_texture(value : text)
  IF value == sky_texture THEN RETURN
  sky_texture = value
  sky_texture_small = value + "#small"        # the low-resolution companion
  descriptor.on_device_create(self)           # re-create the device resources

FUNCTION set_clouds_texture(value : text)
  IF value == clouds_texture THEN RETURN
  clouds_texture = value
  descriptor.on_device_create(self)
```

**Invariants** — a sky texture always has a companion named by appending `#small`, and the
name of that companion is derived, never authored. The engine's sky rendering relies on it
existing.

**Notes** — Re-creating device resources from a property setter is the sharpest thing in
this module: a grid cell edit reaches down to the graphics device mid-session. It is what
makes texture authoring interactive, and it is the reason this file's tier floor is the
device tier. A rebuild must keep the *effect* — the view updates when a texture name
changes — and is free to defer the re-creation to the next frame boundary, which the
original does not.

The `#small` suffix names a variant the texture system resolves; the convention is the
texture system's, not this file's, and nothing here explains what makes it small.

## The angle accessors

```text
FUNCTION sun_altitude() -> real   : RETURN degrees(sun_direction.heading)
FUNCTION set_sun_altitude(v)      : sun_direction.set(heading = radians(v), keep pitch)
FUNCTION sun_longitude() -> real  : RETURN degrees(sun_direction.pitch)
FUNCTION set_sun_longitude(v)     : sun_direction.set(keep heading, pitch = radians(v))
# sky_rotation, clouds_rotation and wind_direction follow the same pattern
```

**Notes** — The sun's direction is stored as a heading-and-pitch pair, and the editor
calls them altitude and longitude. The mapping is: **altitude is the heading component,
longitude is the pitch component** — which is the reverse of what the words suggest, and
is how the shipping cycle files are authored. A rebuild must preserve the mapping or every
existing cycle's sun moves.

Setting one component reads both and writes both, because the pair is stored as a
direction rather than as two angles; there is no way to write one without the other.

## `lerp` — what the interpolated keyframe reports

```text
FUNCTION lerp(parent, from : Time, to : Time, t : real, modifier, power)
  # 1. recompute the clock, unrolling the midnight wrap exactly as engine_impl does
  start = environment.current[0].exec_time
  stop  = environment.current[1].exec_time
  now   = environment.game_time
  IF start >= stop
    IF now >= start THEN clamp(now, start, SECONDS_PER_DAY) ELSE clamp(now, 0, stop)
    IF now <= stop THEN now = now + SECONDS_PER_DAY
    stop = stop + SECONDS_PER_DAY
  ELSE
    clamp(now, start, stop)

  # 2. the interpolated keyframe's identifier IS the clock
  identifier = format now AS HH:MM:SS, wrapped into one day

  # 3. names do not interpolate: take them from the keyframe being blended FROM
  ambient                = from.ambient
  clouds_texture         = from.clouds_texture
  sky_texture            = from.sky_texture
  sun                    = from.sun
  thunderbolt_collection = from.thunderbolt_collection

  # 4. everything numeric interpolates as the engine already does
  base_keyframe.lerp(parent, from, to, t, modifier, power)
```

**Contract** — runs once per frame on the interpolated keyframe. Produces a keyframe that
reads, in the grid, as "what the engine is using right now".

**Invariants** — the clock is recomputed here rather than passed in, and must agree with
the position [`engine_impl.cpp`](engine_impl.cpp.md) computes from the same three values.
Both unroll the midnight wrap the same way; if a rebuild changes one it must change both,
which is the argument for factoring the unroll out once.

**Notes** — Two decisions, both about what "interpolated" means for a field that is not a
number.

**The identifier carries the clock.** This is why
[`engine_impl.cpp`](engine_impl.cpp.md) reads the time of day as a field rather than
formatting it, and why the clipboard's "copy the current blend" lands a new keyframe at
exactly the moment the author was looking at. A rebuild could carry the clock in its own
field; what must survive is that **the interpolated keyframe knows the instant it
represents.**

**Names snap to the source keyframe rather than blending.** There is no meaningful
midpoint between two sky textures or two ambient records, so the blend takes the one it is
coming *from* and holds it until the next keyframe takes over. That makes the ambient
sound set, the sun's flare and the thunderbolt set change discontinuously at keyframe
boundaries, which is the intended behaviour and is visible to the author in the middle
grid.
