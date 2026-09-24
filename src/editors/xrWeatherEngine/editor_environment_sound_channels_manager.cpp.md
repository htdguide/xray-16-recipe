# src/editors/xrWeatherEngine/editor_environment_sound_channels_manager.cpp

> Owns the sound-channel records: the continuous background layers an ambient mixes together.

**Needs** — [`editor_environment_sound_channels_manager.hpp`](editor_environment_sound_channels_manager.hpp.md) · [`editor_environment_sound_channels_channel.hpp`](editor_environment_sound_channels_channel.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`property_collection.hpp`](property_collection.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md) · [`xrCore/LocatorAPI.h`](../../xrCore/LocatorAPI.h.md)
**Used by** — [`editor_environment_sound_channels_manager.hpp`](editor_environment_sound_channels_manager.hpp.md)
**Tier floor** — T2: it owns records and one configuration file.

## Purpose

The same cached-list-plus-editable-collection shape as every other sub-manager, over the
sound-channels file.

## State

See [`editor_environment_sound_channels_manager.hpp`](editor_environment_sound_channels_manager.hpp.md).

## `load` and `save`

```text
FUNCTION load()
  REQUIRE channels IS empty
  config = read configuration AT "$game_config$/environment/sound_channels.ltx"
  FOR EACH section IN config
    channel = new Channel(self, section.name)
    channel.load(config)
    channel.register_with(collection)
    append channel TO channels

FUNCTION save()
  config = new configuration AT the same path, write-on-close
  FOR EACH channel IN channels
    channel.save(INTO config)
```

## `fill`, `channels_ids`, `unique_id`

```text
FUNCTION fill(holder)
  holder.add_property("sound channels", group "ambients", collection)

FUNCTION channels_ids() -> list<text>   # cached; rebuilt when changed is set, natural order
FUNCTION unique_id(id) -> text          # accept if free, else id + first free counter
```

## Element factory

```text
FUNCTION create() -> PropertyHolder
  channel = new Channel(self, generate_unique_id("sound_channel_unique_id_"))
  channel.register_with(this collection)
  RETURN channel.property_holder()

FUNCTION display_name(index) -> text
  RETURN channels[index].id
```
