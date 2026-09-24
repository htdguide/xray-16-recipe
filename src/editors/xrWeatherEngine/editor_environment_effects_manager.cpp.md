# src/editors/xrWeatherEngine/editor_environment_effects_manager.cpp

> Owns the effect records: the occasional bursts — a particle system, a sound, a gust of wind — that an ambient fires.

**Needs** — [`editor_environment_effects_manager.hpp`](editor_environment_effects_manager.hpp.md) · [`editor_environment_effects_effect.hpp`](editor_environment_effects_effect.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`property_collection.hpp`](property_collection.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md) · [`xrCore/LocatorAPI.h`](../../xrCore/LocatorAPI.h.md)
**Used by** — [`editor_environment_effects_manager.hpp`](editor_environment_effects_manager.hpp.md)
**Tier floor** — T2: it owns records and one configuration file.

## Purpose

The effects file is a flat list of named bursts; ambients reference them by name. This
manager is the same cached-list-plus-editable-collection shape as every other sub-manager,
over that one file.

## State

See [`editor_environment_effects_manager.hpp`](editor_environment_effects_manager.hpp.md).

## `load` and `save`

```text
FUNCTION load()
  REQUIRE effects IS empty
  config = read configuration AT "$game_config$/environment/effects.ltx"
  FOR EACH section IN config
    record = new Effect(self, section.name)
    record.load(config)
    record.register_with(collection)
    append record TO effects

FUNCTION save()
  config = new configuration AT the same path, write-on-close
  FOR EACH record IN effects
    record.save(INTO config)
```

**Contract** — one record per section; the section name is the effect's name. Reading and
writing address the same path, unlike the ambients manager.

**Notes** — This manager opens its own configuration rather than taking the environment's,
which is why it can load before ambients do. The ordering constraint runs the other way:
ambients need effects to exist.

## `fill`, `effects_ids`, `unique_id`

```text
FUNCTION fill(holder)
  holder.add_property("effects", group "ambients", collection)

FUNCTION effects_ids() -> list<text>     # cached; rebuilt when changed is set, natural order
FUNCTION unique_id(id) -> text           # accept if free, else id + first free counter
```

**Notes** — The effect list is filed under the *ambients* group in the grid, not a group of
its own, because an author reaches effects through the ambient that fires them. Sound
channels are grouped the same way. That grouping is the editor's one piece of information
architecture and it is right: the model has seven files but only four things an author
thinks about — the day, the sounds, the storms, and the levels.

## Element factory

```text
FUNCTION create() -> PropertyHolder
  record = new Effect(self, generate_unique_id("effect_unique_id_"))
  record.register_with(this collection)
  RETURN record.property_holder()

FUNCTION display_name(index) -> text
  RETURN effects[index].id
```

**Notes** — A new effect is created with every field at its default — no particle system,
no sound, zero life time — and saving it in that state writes a record the game will load
and fire to no visible effect. Nothing validates it.
