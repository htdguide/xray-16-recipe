# src/editors/xrWeatherEngine/editor_environment_sound_channels_manager.hpp

> Declares the owner of the sound-channel records — the continuous background layers an ambient mixes.

**Needs** — [`editor_environment_sound_channels_manager.cpp`](editor_environment_sound_channels_manager.cpp.md) · [`property_collection_forward.hpp`](property_collection_forward.hpp.md) · [`Common/Noncopyable.hpp`](../../Common/Noncopyable.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md)
**Used by** — [`editor_environment_ambients_sound_id.cpp`](editor_environment_ambients_sound_id.cpp.md) · [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) · [`editor_environment_sound_channels_channel.cpp`](editor_environment_sound_channels_channel.cpp.md) · [`editor_environment_sound_channels_manager.cpp`](editor_environment_sound_channels_manager.cpp.md)
**Tier floor** — T2: it owns the records and one configuration file.

## Purpose

Declares the surface implemented in
[`editor_environment_sound_channels_manager.cpp`](editor_environment_sound_channels_manager.cpp.md).

## State

```text
RECORD SoundChannelsManager
  channels    : list<Channel>
  collection  : PropertyCollection
  channel_ids : list<text>      # cached, sorted; rebuilt when changed is set
  changed     : bool
```

It is the only sub-manager that needs no reference to the weather system — a sound channel
browses nothing but files.

## Exported units

- **`load` / `save`** — read and write the sound-channels file.
- **`fill`** — add the channel list to the grid, under the ambients group.
- **`channels_ids`** — the sorted record names, for an ambient's sound-channel picker.
- **`unique_id`** — the renaming rule.
