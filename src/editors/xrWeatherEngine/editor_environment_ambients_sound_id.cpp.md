# src/editors/xrWeatherEngine/editor_environment_ambients_sound_id.cpp

> One entry in an ambient's sound-channel list: a name, constrained to the channels the model defines.

**Needs** — [`editor_environment_ambients_sound_id.hpp`](editor_environment_ambients_sound_id.hpp.md) · [`editor_environment_sound_channels_manager.hpp`](editor_environment_sound_channels_manager.hpp.md) · [`ide.hpp`](ide.hpp.md)
**Used by** — [`editor_environment_ambients_sound_id.hpp`](editor_environment_ambients_sound_id.hpp.md)
**Tier floor** — T3: a name and a picker.

## Purpose

The sound-channel twin of
[`effect_id`](editor_environment_ambients_effect_id.cpp.md), and identical in every
respect except which manager supplies the options. The two files exist separately because
each binds to a different source; a rebuild parameterises one type over the source and
deletes the other.

## State

See [`editor_environment_ambients_sound_id.hpp`](editor_environment_ambients_sound_id.hpp.md).

## `fill`

```text
FUNCTION fill(collection : PropertyCollection)
  property_holder = editor.create_property_holder(id, collection, owner = self)
  property_holder.add_property(
      "sound channel", group "properties",
      value bound to id,
      options pulled from sounds_manager.channels_ids,
      editor = combo box, free text not allowed)
```

**Contract** — one row: a combo box over every sound-channel name the model defines.
Options are pulled at paint time. Typing is disallowed, for the reason given in
[`effect_id`](editor_environment_ambients_effect_id.cpp.md).
