# src/xrCore/xr_ini.h

> Declares the configuration (`ltx`) surface implemented in [`xr_ini.cpp`](xr_ini.cpp.md), plus the generic typed-read helpers every caller uses.

**Needs** — [`xr_ini.cpp`](xr_ini.cpp.md) · [`xrstring.h`](xrstring.h.md) · [`fastdelegate.h`](fastdelegate.h.md) · [`clsid.h`](clsid.h.md) · [`_flags.h`](_flags.h.md) · [`_color.h`](_color.h.md) · [`_vector2.h`](_vector2.h.md) · [`_vector3d.h`](_vector3d.h.md) · [`_vector4.h`](_vector4.h.md) · [`../xrCommon/xr_vector.h`](../xrCommon/xr_vector.h.md)
**Used by** — [`dx113DFluidData.cpp`](../Layers/xrRenderDX11/3DFluid/dx113DFluidData.cpp.md) · [`editor_environment_ambients_ambient.cpp`](../editors/xrWeatherEngine/editor_environment_ambients_ambient.cpp.md) · [`editor_environment_ambients_manager.cpp`](../editors/xrWeatherEngine/editor_environment_ambients_manager.cpp.md) · [`editor_environment_effects_effect.cpp`](../editors/xrWeatherEngine/editor_environment_effects_effect.cpp.md) · [`editor_environment_effects_manager.cpp`](../editors/xrWeatherEngine/editor_environment_effects_manager.cpp.md) · [`editor_environment_levels_manager.cpp`](../editors/xrWeatherEngine/editor_environment_levels_manager.cpp.md) · [`editor_environment_sound_channels_channel.cpp`](../editors/xrWeatherEngine/editor_environment_sound_channels_channel.cpp.md) · [`editor_environment_sound_channels_manager.cpp`](../editors/xrWeatherEngine/editor_environment_sound_channels_manager.cpp.md) · [`editor_environment_suns_blend.cpp`](../editors/xrWeatherEngine/editor_environment_suns_blend.cpp.md) · [`editor_environment_suns_flares.cpp`](../editors/xrWeatherEngine/editor_environment_suns_flares.cpp.md) · [`editor_environment_suns_gradient.cpp`](../editors/xrWeatherEngine/editor_environment_suns_gradient.cpp.md) · [`editor_environment_suns_manager.cpp`](../editors/xrWeatherEngine/editor_environment_suns_manager.cpp.md) · [`editor_environment_suns_sun.cpp`](../editors/xrWeatherEngine/editor_environment_suns_sun.cpp.md) · [`editor_environment_thunderbolts_collection.cpp`](../editors/xrWeatherEngine/editor_environment_thunderbolts_collection.cpp.md) · _and 31 more_
**Tier floor** — T3: a declaration surface over a text format; nothing here constrains the language.

## Purpose

Declares the configuration type, its item and section records, and the read/write surface described in full in [`xr_ini.cpp`](xr_ini.cpp.md). Its own contribution is the *shape of the read call*, which is what the rest of the engine actually sees: a family of four read styles built on two primitives (`line_exists` and a typed read).

## Exported units

- **`Item`** — a key/value pair, both interned strings, both nullable.
- **`Section`** — a name plus its sorted item list, with a probe that returns the value for a key without failing.
- **`Config`** — the parsed file. Creation, release, load-from-stream, load-from-path, save.
- **Typed reads** — one per type: string (quotes kept), string with quotes stripped, signed and unsigned integers of every width, real, float colour, packed colour, 2/3/4-element integer and real vectors, bool, class identifier, token-by-table, and read-by-position.
- **Typed writes** — the mirror set, each formatting to text and delegating to the string write.
- **Read styles** — the four ways a caller may ask for a value, and choosing between them *is* the error-handling policy of the whole engine:
  - *read* — fatal if the key is missing. The default, and why the shipped configuration set must parse completely.
  - *read-if-exists* — probe first, return a caller-supplied default, or report whether it was found.
  - *read-if-exists with a fallback key* — try one key, then a second spelling; optionally force the fatal path when neither is present, so the crash message names the *preferred* key. This exists because several shipped settings were renamed between games.
  - *try-read* — for tuple types only: reports how many components actually parsed.
- **Predicate type for include vetoing** — lets a caller refuse to follow an include by path.
- **Section name for this engine's own settings** — the one section name reserved for additions that the original engine never wrote.
- **Three global configurations** — main settings, authentication settings, and this engine's own settings.
