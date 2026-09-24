# src/editors/xrWeatherEngine/editor_environment_suns_manager.cpp

> Owns the sun records, and is the one sub-manager the editor deliberately refuses to save.

**Needs** — [`editor_environment_suns_manager.hpp`](editor_environment_suns_manager.hpp.md) · [`editor_environment_suns_sun.hpp`](editor_environment_suns_sun.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`property_collection.hpp`](property_collection.hpp.md) · [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md) · [`Common/object_broker.h`](../../Common/object_broker.h.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)
**Used by** — [`editor_environment_suns_manager.hpp`](editor_environment_suns_manager.hpp.md)
**Tier floor** — T2: it owns records and one configuration file.

## Purpose

The same sub-manager shape as its siblings, over the suns file — with two deviations that
matter more than the shape: **its saved output is not trusted**, and **its name list
contains an empty name.**

## State

See [`editor_environment_suns_manager.hpp`](editor_environment_suns_manager.hpp.md).

## `load` and `save`

```text
FUNCTION load()
  config = read configuration AT "$game_config$/environment/suns.ltx"
  FOR EACH section IN config
    add(config, section.name)

FUNCTION add(config, section : text)
  REQUIRE no sun already carries section
  sun = new Sun(self, section)
  sun.load(config)
  sun.register_with(collection)
  append sun TO suns

FUNCTION save()
  config = new configuration AT the same path, write-on-close
  FOR EACH sun IN suns
    sun.save(INTO config)
```

**Notes** — `save` is defined and correct for what it writes, and **nothing calls it**. The
environment's save omits it with a stated reason: the editor reads only part of a sun's
authored record, so writing would silently drop the rest. See
[`editor_environment_suns_sun.cpp`](editor_environment_suns_sun.cpp.md) for exactly which
fields survive and which do not, and
[`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) for the general rule.

A rebuild has two honest options: complete the sun surface and then save it, or delete the
save. Keeping a save that is never called is the one thing to avoid, because the next
maintainer will call it.

## `suns_ids` — the list with an empty name in it

```text
FUNCTION suns_ids() -> list<text>
  IF NOT changed THEN RETURN cache
  changed = false
  cache = [ "" ] + every sun's id
  sort cache BY natural order
  RETURN cache
```

**Contract** — the sorted sun names, **with an empty name prepended** before sorting.

**Notes** — The empty entry is how a keyframe says "no sun flare". Without it the keyframe's
combo box — which disallows free text — would make the choice permanent once made. The
thunderbolt-collection list does the same thing for the same reason; the ambient and
effect lists do not, because a keyframe must always have an ambient and an ambient's effect
list can simply be empty.

## `unique_id` and `fill`

```text
FUNCTION fill(holder)
  holder.add_property("suns", group "suns", collection)

FUNCTION unique_id(id) -> text          # accept if free, else id + first free counter
```

**Notes** — The row's description reads "this option is responsible for sound channels",
copied from a sibling and never corrected. Cosmetic, and worth fixing in a rebuild only
because the descriptions are the model's documentation.

## `get_flare` — a stub

Declared to resolve a sun name to its flare description; the body is commented out and it
returns nothing. Nothing calls it. The flare a keyframe uses is resolved through the
lens-flare library instead, by name, in
[`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md).

## Element factory

```text
FUNCTION create() -> PropertyHolder
  sun = new Sun(self, generate_unique_id("sun_unique_id_"))
  sun.register_with(this collection)
  RETURN sun.property_holder()

FUNCTION display_name(index) -> text
  RETURN suns[index].id
```

**Notes** — Since the suns file is never written, a sun created here exists only for the
session. It can be selected by a keyframe, and that keyframe *will* be saved with the
name — leaving a cycle file that references a sun no file defines. A rebuild that keeps the
no-save decision should also stop offering the add command on this list.
