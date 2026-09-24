# src/editors/xrWeatherEngine/editor_environment_effects_effect.cpp

> One effect record: what fires, where, for how long, and how hard it blows.

**Needs** — [`editor_environment_effects_effect.hpp`](editor_environment_effects_effect.hpp.md) · [`editor_environment_effects_manager.hpp`](editor_environment_effects_manager.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md) · [Seam: Audio device](../../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — [`editor_environment_effects_effect.hpp`](editor_environment_effects_effect.hpp.md)
**Tier floor** — T1: a grid edit destroys and re-creates an audio source.

## Purpose

An effect is one occasional event in the world's background: a distant thunder roll with a
gust behind it, a flock of birds, a creak. This file fixes its on-disk shape and its
authoring page.

## State

See [`editor_environment_effects_effect.hpp`](editor_environment_effects_effect.hpp.md).

## The effect record on disk

```text
RECORD EffectSection                    # one configuration section per effect
  section name          : text
  life_time             : int           # milliseconds
  offset                : vec3          # metres
  particles             : text          # a particle-system name
  sound                 : text          # a sound path, extension dropped
  wind_gust_factor      : real
  wind_blast_in_time    : real          # seconds
  wind_blast_out_time   : real          # seconds
  wind_blast_strength   : real
  wind_blast_longitude  : real          # degrees on disk, a direction in memory
```

**Invariants** — every field is required on read; there are no defaults, so a section
missing one fails to load. The blast longitude is the direction's heading component; its
other component is always zero, so a blast is horizontal by construction.

**Notes** — **This layout is frozen.** Note that reading and writing are symmetric here,
unlike the keyframe: this record is small enough that the editor owns all of it, which is
why the effects file is safe to rewrite.

## `load` and `save`

```text
FUNCTION load(config)
  read every field FROM the section named id
  wind_blast_direction = direction(heading = radians(config.real(id, "wind_blast_longitude")),
                                   pitch = 0)

FUNCTION save(config)
  write every field INTO the section named id
  write "wind_blast_longitude" = degrees(wind_blast_direction.heading)
```

## `fill` — the effect's ten rows

```text
GROUP properties : id (text, filtered through the manager's naming rule)
                   life time (integer, milliseconds)
                   offset (three-component vector)
                   particles (chosen from the model's particle-system names, tree picker)
                   sound (file browser, .ogg, extension dropped)
                   wind gust factor (real)
                   wind blast strength (real)
                   wind blast start time (real, 0..1000)
                   wind blast stop time (real, 0..1000)
                   wind blast longitude (real, -360..360 degrees)
```

**Notes** — The particle row uses a **tree** picker rather than a combo box, because
particle-system names are paths with a folder structure and there are hundreds of them; the
shader rows elsewhere in this module do the same for the same reason. The choice between
the two pickers across this module is consistently "combo box when the list is tens, tree
when it is hundreds and hierarchical".

The life time row binds to the field directly, reinterpreting an unsigned count as a signed
integer so the grid can show it. Harmless at these magnitudes and incidental.

## The sound setter

```text
FUNCTION set_sound(value : text)
  sound_name = value
  release the current audio source
  create an audio source FROM value, as an effect, positioned in the world
```

**Contract** — writing the sound name destroys the existing audio source and creates a new
one immediately, so the author hears the change without reloading.

**Notes** — Unlike every other setter in this module, this one does **not** return early
when the value is unchanged, so re-selecting the same sound still cycles the audio source.
Harmless, and inconsistent with its neighbours.

The audio source is created as a positioned effect source rather than a stream, which is
what lets the offset field mean anything; see
[Seam: Audio device](../../../SYSTEM-REQUIREMENTS.md#seam-audio-device).
