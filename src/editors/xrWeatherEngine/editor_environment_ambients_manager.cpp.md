# src/editors/xrWeatherEngine/editor_environment_ambients_manager.cpp

> Owns the ambient records: the named bundles of background sound channels and occasional effects that a keyframe selects by name.

**Needs** — [`editor_environment_ambients_manager.hpp`](editor_environment_ambients_manager.hpp.md) · [`editor_environment_ambients_ambient.hpp`](editor_environment_ambients_ambient.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`property_collection.hpp`](property_collection.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)
**Used by** — [`editor_environment_ambients_manager.hpp`](editor_environment_ambients_manager.hpp.md)
**Tier floor** — T2: it owns records and one configuration file.

## Purpose

An *ambient* is what a keyframe names when it says "this is what the world sounds like now":
a set of sound channels that play continuously and a set of effects that fire
occasionally. This manager owns the whole set of them.

## State

See [`editor_environment_ambients_manager.hpp`](editor_environment_ambients_manager.hpp.md).

## `load` and `save`

```text
FUNCTION load()
  REQUIRE ambients IS empty
  FOR EACH section IN environment.ambients_config
    record = new Ambient(self, section.name)
    record.load(ambients_config, sound_channels_config, effects_config, section.name)
    record.register_with(collection)
    append record TO ambients

FUNCTION save()
  config = new configuration AT "$game_config$/environment/ambients.ltx", write-on-close
  FOR EACH record IN ambients
    record.save(INTO config)
```

**Contract** — one record per section of the ambients configuration. Loading reads three
configurations at once, because an ambient's children are defined in the sound-channel and
effect files and must be resolved while the record is being read.

**Notes** — The configurations are the *environment's own already-open* ones, not files
this manager opens — which is why ambients must load after the effects and sound-channel
managers, and why the load order in
[`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) is not arbitrary.

Saving writes to a path this file names literally, while loading takes whatever the
environment had open. That asymmetry means an installation that mounts its ambients from
somewhere else reads from there and writes to here. A rebuild should carry the path
alongside the loaded configuration.

## `fill`, `ambients_ids`, `unique_id`

```text
FUNCTION fill(holder)
  holder.add_property("ambients", collection)

FUNCTION ambients_ids() -> list<text>     # cached; rebuilt when changed is set, natural order
FUNCTION unique_id(id) -> text            # accept if free, else id + first free counter
```

Identical in shape to the cached list and renaming rule described in
[`editor_environment_weathers_manager.cpp`](editor_environment_weathers_manager.cpp.md).

## `get_ambient`

```text
FUNCTION get_ambient(id : text) -> Ambient
  FOR EACH record IN ambients
    IF record.id == id THEN RETURN record
  UNREACHABLE                    # an unknown name is a programming error, not a data error
```

**Contract** — resolves a name to a record. An unknown name is treated as impossible.

**Notes** — This is the lookup the base weather system reaches when loading a keyframe's
`ambient` field, through the factory hook in
[`editor_environment_manager.cpp`](editor_environment_manager.cpp.md). The hard failure is
consistent with that: inside the editor every ambient a keyframe can name has already been
built. It is *not* safe against a hand-edited cycle file that names an ambient the
ambients file does not define — the game tolerates that case, the editor does not. A
rebuild should report it as a data error.

## Element factory

```text
FUNCTION create() -> PropertyHolder
  record = new Ambient(self, generate_unique_id("ambient_unique_id_"))
  record.register_with(this collection)
  RETURN record.property_holder()

FUNCTION display_name(index) -> text
  RETURN ambients[index].id
```
