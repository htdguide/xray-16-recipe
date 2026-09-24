# src/editors/xrWeatherEngine/property_collection_forward.hpp

> Announces that the editable-list adapter exists, so headers can hold a reference to one without pulling in its implementation.

**Needs** — [`property_collection.hpp`](property_collection.hpp.md)
**Used by** — [`editor_environment_ambients_ambient.hpp`](editor_environment_ambients_ambient.hpp.md) · [`editor_environment_ambients_manager.hpp`](editor_environment_ambients_manager.hpp.md) · [`editor_environment_effects_manager.hpp`](editor_environment_effects_manager.hpp.md) · [`editor_environment_sound_channels_channel.hpp`](editor_environment_sound_channels_channel.hpp.md) · [`editor_environment_sound_channels_manager.hpp`](editor_environment_sound_channels_manager.hpp.md) · [`editor_environment_suns_flares.hpp`](editor_environment_suns_flares.hpp.md) · [`editor_environment_suns_manager.hpp`](editor_environment_suns_manager.hpp.md) · [`editor_environment_thunderbolts_collection.hpp`](editor_environment_thunderbolts_collection.hpp.md) · [`editor_environment_thunderbolts_manager.hpp`](editor_environment_thunderbolts_manager.hpp.md) · [`editor_environment_weathers_manager.hpp`](editor_environment_weathers_manager.hpp.md) · [`editor_environment_weathers_weather.hpp`](editor_environment_weathers_weather.hpp.md)
**Tier floor** — T4: a compile-time declaration with no content.

## Purpose

Incidental. C++ needs a name declared before it can be used as a pointer type; almost
every class in this module holds a pointer to an editable-list adapter, and none of them
needs its body. A rebuild deletes this file.

## State

`Stateless.`
