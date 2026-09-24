# src/editors/xrWeatherEngine/editor_environment_sound_channels_source.cpp

> One sound file in a channel's pool: a path the author picks with a file dialog.

**Needs** — [`editor_environment_sound_channels_source.hpp`](editor_environment_sound_channels_source.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`ide.hpp`](ide.hpp.md) · [Seam: Windowing and input](../../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`editor_environment_sound_channels_source.hpp`](editor_environment_sound_channels_source.hpp.md)
**Tier floor** — T3: a path and a file browser.

## Purpose

The counterpart of
[`effect_id`](editor_environment_ambients_effect_id.cpp.md) for sound files, and the one
difference is the whole point: an effect reference is picked from a *list the model
defines*, while a sound source is picked from *the filesystem*. There is no registry of
sound files, so the picker is a file dialog.

## State

See [`editor_environment_sound_channels_source.hpp`](editor_environment_sound_channels_source.hpp.md).

## `fill`

```text
FUNCTION fill(collection : PropertyCollection)
  property_holder = editor.create_property_holder(path, collection, owner = self)
  property_holder.add_property(
      "sound", group "properties",
      value bound to path,
      browse: extension ".ogg", mask "Sound files (*.ogg)|*.ogg",
              start folder = resolve("$game_sounds$"), caption "Select sound...",
      free text not allowed, extension dropped from the result)
```

**Contract** — one row: a file browser rooted at the game's sound folder, accepting one
container format, with the extension removed from what is stored.

**Notes** — **Dropping the extension is not cosmetic.** The engine names sounds without
one and resolves the container itself, so storing `ambient\wind_01` rather than
`ambient\wind_01.ogg` is what makes the reference resolvable. Every browsable-file row in
this module drops the extension except the thunderbolt's lighting-model row, which keeps
it — see
[`editor_environment_thunderbolts_thunderbolt.cpp`](editor_environment_thunderbolts_thunderbolt.cpp.md).

Disallowing free text means a sound can only be chosen from disk, so a path that does not
exist cannot be entered — but a path whose file is later deleted stays.
