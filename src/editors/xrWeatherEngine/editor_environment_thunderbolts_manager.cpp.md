# src/editors/xrWeatherEngine/editor_environment_thunderbolts_manager.cpp

> Owns the thunderbolts, the named sets a keyframe draws from, and the eight world-wide parameters that place a strike in the sky.

**Needs** — [`editor_environment_thunderbolts_manager.hpp`](editor_environment_thunderbolts_manager.hpp.md) · [`editor_environment_thunderbolts_thunderbolt.hpp`](editor_environment_thunderbolts_thunderbolt.hpp.md) · [`editor_environment_thunderbolts_collection.hpp`](editor_environment_thunderbolts_collection.hpp.md) · [`editor_environment_thunderbolts_thunderbolt_id.hpp`](editor_environment_thunderbolts_thunderbolt_id.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`property_collection.hpp`](property_collection.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`xrEngine/IGame_Persistent.h`](../../xrEngine/IGame_Persistent.h.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)
**Used by** — [`editor_environment_thunderbolts_manager.hpp`](editor_environment_thunderbolts_manager.hpp.md)
**Tier floor** — T2: it owns two record sets and writes three configuration files.

## Purpose

Thunderbolts are authored in two layers: individual strikes, and named sets of them. A
keyframe names a set; the engine picks a strike from it at random. On top of that sit eight
numbers that are not per-strike or per-set but per *world* — where in the sky strikes
appear and how they tint the scene. This manager owns all three.

## State

See [`editor_environment_thunderbolts_manager.hpp`](editor_environment_thunderbolts_manager.hpp.md).

## `load` and `save`

```text
FUNCTION load()
  load_thunderbolts()      # "$game_config$/environment/thunderbolts.ltx"
  load_collections()       # "$game_config$/environment/thunderbolt_collections.ltx"

FUNCTION save()
  save_thunderbolts()
  save_collections()
  config = new configuration AT "$game_config$/environment/environment.ltx", write-on-close
  write the eight storm parameters INTO section "environment"
```

**Invariants** — thunderbolts must load before collections, because a collection resolves
each of its entries to a strike record as it reads.

**Notes** — Saving writes a *third* file this manager never reads: the world's environment
file. That asymmetry is where the eight storm parameters live — they are loaded by the
engine's own environment loader into fields on the weather system, and this manager only
edits and re-emits them. **The whole environment file is rewritten to update eight keys**,
so any other key in it is lost. That is the same hazard as the level catalogues, and the
same fix applies: write back in place.

## The storm parameters on disk

```text
RECORD EnvironmentSection               # section "environment" in the world's environment file
  altitude          : real   # degrees on disk — where strikes appear above the horizon
  delta_longitude   : real   # degrees on disk — the spread around the view direction
  min_dist_factor   : real   # 0..0.95 — nearest strike, as a fraction of the far plane
  tilt              : real   # degrees on disk, 15..30 — how far a bolt leans
  second_probability : real  # 0..1 — chance of a second strike following the first
  sky_color         : real   # 0..1 — how much a strike brightens the sky
  sun_color         : real   # 0..1 — how much it brightens the sun term
  fog_color         : real   # 0..1 — how much it brightens the fog
```

**Invariants** — angles are degrees on disk and radians in memory, as everywhere in this
module. The three colour factors are scalars, not colours: a strike does not tint, it
brightens.

**Notes** — **The bounds are the specification** and exist only in the grid rows. The tilt
bound is the strangest and the most informative: `15..30` degrees is not a sanity range,
it is the range in which a bolt looks like a bolt. The minimum distance bound stops just
short of one so a strike is never placed exactly at the far plane, where it would clip.

The altitude field is a two-component quantity in memory and the editor reads and writes
only the first; the original marks this in three places as work to be done. So the second
component survives a save only because the engine loader reads it from a file this manager
overwrites — which it does not preserve. **A rebuild must read and write both components,
or the vertical spread of strikes is silently lost on the first save.** This is the single
most damaging known defect in the module's write path.

## `fill`

```text
FUNCTION fill(holder)
  # the eight storm parameters, all in group "thunderbolts"
  altitude          (bound to accessors: degrees <-> radians, -360..360)
  delta longitude   (bound to accessors: degrees <-> radians, -360..360)
  minimum distance factor (bound to the field, 0..0.95)
  tilt              (bound to accessors: degrees <-> radians, 15..30)
  second probability (bound to the field, 0..1)
  sky color         (bound to the field, 0..1)
  sun color         (bound to the field, 0..1)
  fog color         (bound to the field, 0..1)
  # then the two lists
  holder.add_property("thunderbolt collections", collections_collection)
  holder.add_property("thunderbolts", thunderbolt_collection)
```

**Notes** — The three angle rows bind through accessors for the unit conversion; the five
scalar rows bind to the field. Same split as everywhere: accessors where there is a side
effect, fields where there is not.

## The two cached name lists

```text
FUNCTION thunderbolts_ids() -> list<text>
  IF NOT thunderbolts_changed THEN RETURN cache
  rebuild cache FROM thunderbolts
  sort cache BY natural order
  RETURN cache

FUNCTION collections_ids() -> list<text>
  IF NOT collections_changed THEN RETURN cache
  cache = [ "" ] + every collection's id
  sort cache BY natural order
  RETURN cache
```

**Notes** — Two defects, both real.

**Neither clears its change flag.** Every other cached list in this module sets the flag to
false after rebuilding; these two do not, so once anything changes they rebuild on every
request — which for a grid repaint is every frame. Correct, wasteful, and trivially fixed.

The collections list prepends an empty name so a keyframe can select "no thunderbolts",
the same device the suns list uses. The thunderbolt list does not, because a collection
entry must name a real strike.

## Resolving a name

```text
FUNCTION description(config, section) -> Thunderbolt
  FOR EACH bolt IN thunderbolts
    IF bolt.id == section THEN RETURN bolt
  UNREACHABLE

FUNCTION get_collection(section) -> Collection
  FOR EACH set IN collections
    IF set.id == section THEN RETURN set
  UNREACHABLE
```

**Contract** — both are the factory hooks the base weather system calls, reached through
[`editor_environment_manager.cpp`](editor_environment_manager.cpp.md); both ignore the
configuration argument and return the already-built editable record. An unknown name is
treated as impossible, with the same caveat noted for ambients.

## Element factories

```text
FUNCTION create_thunderbolt() -> PropertyHolder
  bolt = new Thunderbolt(self, generate_unique_id("thunderbolt_unique_id_"))
  bolt.register_with(this collection)
  RETURN bolt.property_holder()

FUNCTION create_collection() -> PropertyHolder
  set = new Collection(self, generate_unique_id("thunderbolt_collection_unique_id_"))
  set.register_with(this collection)
  RETURN set.property_holder()
```

**Notes** — A thunderbolt created this way has no gradients — they are built only by the
base loader's factory hooks, which run on load. Saving it writes a record whose gradient
keys are missing, and reloading it then fails. **The add command on the thunderbolt list is
not safe**, and a rebuild must either construct both gradients in the constructor or
refuse the command. See
[`editor_environment_thunderbolts_thunderbolt.cpp`](editor_environment_thunderbolts_thunderbolt.cpp.md).
