# src/Common/Noncopyable.hpp

> A marker a type inherits to declare that it has identity, not value — copying one is a bug.

**Needs** — _(none)_
**Used by** — [`editor_environment_ambients_ambient.hpp`](../editors/xrWeatherEngine/editor_environment_ambients_ambient.hpp.md) · [`editor_environment_ambients_effect_id.hpp`](../editors/xrWeatherEngine/editor_environment_ambients_effect_id.hpp.md) · [`editor_environment_ambients_manager.hpp`](../editors/xrWeatherEngine/editor_environment_ambients_manager.hpp.md) · [`editor_environment_ambients_sound_id.hpp`](../editors/xrWeatherEngine/editor_environment_ambients_sound_id.hpp.md) · [`editor_environment_effects_effect.hpp`](../editors/xrWeatherEngine/editor_environment_effects_effect.hpp.md) · [`editor_environment_effects_manager.hpp`](../editors/xrWeatherEngine/editor_environment_effects_manager.hpp.md) · [`editor_environment_levels_manager.hpp`](../editors/xrWeatherEngine/editor_environment_levels_manager.hpp.md) · [`editor_environment_sound_channels_channel.hpp`](../editors/xrWeatherEngine/editor_environment_sound_channels_channel.hpp.md) · [`editor_environment_sound_channels_manager.hpp`](../editors/xrWeatherEngine/editor_environment_sound_channels_manager.hpp.md) · [`editor_environment_sound_channels_source.hpp`](../editors/xrWeatherEngine/editor_environment_sound_channels_source.hpp.md) · [`editor_environment_suns_blend.hpp`](../editors/xrWeatherEngine/editor_environment_suns_blend.hpp.md) · [`editor_environment_suns_flare.hpp`](../editors/xrWeatherEngine/editor_environment_suns_flare.hpp.md) · [`editor_environment_suns_flares.hpp`](../editors/xrWeatherEngine/editor_environment_suns_flares.hpp.md) · [`editor_environment_suns_gradient.hpp`](../editors/xrWeatherEngine/editor_environment_suns_gradient.hpp.md) · _and 24 more_
**Tier floor** — T4: it states an intent about a type. Most languages state it directly, or have it as the default.

## Purpose

Many engine types own something unshareable — a device resource, a slot in a registry, a
position in a scheduler. Copying such a value silently produces two owners of one thing,
which shows up much later as a double release. This file exists so a type can say "I have
identity" in one word and have the compiler enforce it.

## State

Stateless.

## `Noncopyable`

**Contract** — a type mixes this in to declare itself non-copyable and non-assignable. It adds
no fields, no behaviour and no runtime cost; it exists purely so the attempt to copy is
rejected before the program runs.

## Notes

This is the incidental half of a load-bearing idea. The idea — *these objects are
referenced, never duplicated* — survives any rebuild. The mechanism does not: a language
whose aggregates are reference types by default already has it, and a language with move
semantics or affine types expresses it more precisely. What a rebuilder needs to carry
across is the list of types that wear this marker, which is in those types' own twins.
