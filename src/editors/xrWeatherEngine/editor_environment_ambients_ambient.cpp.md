# src/editors/xrWeatherEngine/editor_environment_ambients_ambient.cpp

> One ambient record: which sound channels play continuously, which effects fire occasionally, and how often.

**Needs** — [`editor_environment_ambients_ambient.hpp`](editor_environment_ambients_ambient.hpp.md) · [`editor_environment_ambients_manager.hpp`](editor_environment_ambients_manager.hpp.md) · [`editor_environment_ambients_effect_id.hpp`](editor_environment_ambients_effect_id.hpp.md) · [`editor_environment_ambients_sound_id.hpp`](editor_environment_ambients_sound_id.hpp.md) · [`editor_environment_effects_effect.hpp`](editor_environment_effects_effect.hpp.md) · [`editor_environment_sound_channels_channel.hpp`](editor_environment_sound_channels_channel.hpp.md) · [`property_collection.hpp`](property_collection.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)
**Used by** — [`editor_environment_ambients_ambient.hpp`](editor_environment_ambients_ambient.hpp.md)
**Tier floor** — T2: it owns two editable lists and mirrors them into the engine's resolved lists.

## Purpose

An ambient is a *reference bundle*: it holds no sounds or effects of its own, only the
names of records defined elsewhere, plus the interval at which effects fire. This file
defines its on-disk shape and its grid page.

## State

See [`editor_environment_ambients_ambient.hpp`](editor_environment_ambients_ambient.hpp.md).

## The ambient record on disk

```text
RECORD AmbientSection                     # one configuration section per ambient
  section name       : text               # the ambient's name, as keyframes reference it
  sound_channels     : text               # comma-separated sound-channel names
  effects            : text               # comma-separated effect names
  min_effect_period  : real               # seconds
  max_effect_period  : real               # seconds
```

**Invariants** — the period is written in seconds and held in milliseconds; the conversion
is a factor of a thousand on the way out and is done by the base loader on the way in. The
two lists are comma-separated names, each of which must exist in its own file.

**Notes** — **This layout is frozen**; the game reads it. The list-in-a-string encoding is
the configuration format's only way to express a sequence, and it is what forces the
comma-joining on save. A rebuild using a format with real lists must still emit this one.

## `load`

```text
FUNCTION load(ambients_config, sound_channels_config, effects_config, section)
  REQUIRE section == id
  base_ambient.load(ambients_config, sound_channels_config, effects_config, id)
    # ... which calls back into create_effect and create_sound_channel below
  FOR EACH name IN split(ambients_config.text_or_empty(id, "effects"))
    append new EffectId(effects_manager, name) TO effect_names
  FOR EACH name IN split(ambients_config.text_or_empty(id, "sound_channels"))
    append new SoundId(sounds_manager, name) TO sound_channel_names
```

**Contract** — the base loader reads the periods and resolves the two name lists into
playable records; this file then reads the same two lists *again* as editable name
references. Missing lists default to empty rather than failing.

**Notes** — Reading the lists twice is the price of the base record storing resolutions,
not names. The editable list is what the grid shows and what save writes; the resolved list
is what the engine plays. **They are built from the same text and never reconciled
afterwards** — so an author who removes a name from the grid changes what is saved but not
what is currently playing, until the model is reloaded. That is a real limitation of the
design and a rebuild should either keep one list or re-resolve on edit.

## `save`

```text
FUNCTION save(config)
  config.write_text(id, "sound_channels", join(sound_channel_names, ", "))
  config.write_real(id, "min_effect_period", effect_period.min / 1000)
  config.write_real(id, "max_effect_period", effect_period.max / 1000)
  config.write_text(id, "effects", join(effect_names, ", "))
```

**Notes** — The joined strings are built into a buffer sized by summing the name lengths
plus two characters of separator each, plus one. A rebuild builds a string; the sizing is
incidental. What is load-bearing is the separator — comma-space — and that an empty list
writes an empty value rather than omitting the key.

## The two factory hooks

```text
FUNCTION create_effect(config, id) -> Effect
  effect = new EditableEffect(effects_manager, id)
  effect.load(config)
  effect.register_with(effect_collection)
  RETURN effect

FUNCTION create_sound_channel(config, id, section_to_read_from) -> SoundChannel
  channel = new EditableSoundChannel(sounds_manager, id)
  channel.load(config, section_to_read_from)
  channel.register_with(sound_collection)
  RETURN channel
```

**Contract** — the base loader calls these while resolving the two name lists. They return
editable records, which is the same substitution the environment makes at the level above.

**Notes** — Worth noticing what this produces: an effect record built *here* is registered
with this ambient's own collection, and one built by the effects manager is registered with
that manager's. The same named effect therefore exists as two objects with two grid rows,
edited independently. Whichever was saved last wins. A rebuild should resolve names to a
single shared record; this duplication is the design's weakest seam.

## `fill` — the ambient's five rows

```text
GROUP properties : id (text, filtered through the manager's naming rule)
GROUP effects    : minimum period (integer, milliseconds);
                   maximum period (integer, milliseconds);
                   effects (an editable list of effect names)
GROUP sounds     : sound channels (an editable list of sound-channel names)
```

**Notes** — The two period rows are labelled "in seconds" in their descriptions but bound
directly to the millisecond fields, so the grid shows milliseconds under a label that says
seconds. That is a defect in the original, visible to authors, and a rebuild should bind
them through a converting accessor the way the angle rows do.
