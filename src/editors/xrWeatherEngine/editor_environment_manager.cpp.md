# src/editors/xrWeatherEngine/editor_environment_manager.cpp

> Replaces the engine's weather system with one whose every record is editable, and gathers the name lists the grid's pickers browse.

**Needs** — [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`editor_environment_suns_manager.hpp`](editor_environment_suns_manager.hpp.md) · [`editor_environment_levels_manager.hpp`](editor_environment_levels_manager.hpp.md) · [`editor_environment_effects_manager.hpp`](editor_environment_effects_manager.hpp.md) · [`editor_environment_sound_channels_manager.hpp`](editor_environment_sound_channels_manager.hpp.md) · [`editor_environment_ambients_manager.hpp`](editor_environment_ambients_manager.hpp.md) · [`editor_environment_thunderbolts_manager.hpp`](editor_environment_thunderbolts_manager.hpp.md) · [`editor_environment_weathers_manager.hpp`](editor_environment_weathers_manager.hpp.md) · [`editor_environment_weathers_time.hpp`](editor_environment_weathers_time.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`xrEngine/LightAnimLibrary.h`](../../xrEngine/LightAnimLibrary.h.md) · [`xrEngine/xr_efflensflare.h`](../../xrEngine/xr_efflensflare.h.md) · [`Include/xrRender/particles_systems_library_interface.hpp`](../../Include/xrRender/particles_systems_library_interface.hpp.md) · [`Common/object_broker.h`](../../Common/object_broker.h.md)
**Used by** — [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md)
**Tier floor** — T1: it reads a chunked binary library file to enumerate shader names.

## Purpose

The root of the editor's document model, and the point where "the document" and "the
running world" are decided to be the same object. The engine's weather system builds
authored records from configuration; this one builds *editable* records instead, by
overriding the three factory hooks the base system calls and by owning seven sub-managers
that each cover one authored file set.

## State

See [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md).

## Construction and the load order

```text
FUNCTION construct()
  effects        = new EffectsManager(self)
  sound_channels = new SoundChannelsManager()
  ambients       = new AmbientsManager(self)
  weathers       = new WeathersManager(self)
  suns           = new SunsManager(self)
  levels         = new LevelsManager(weathers)
  thunderbolts   = new ThunderboltsManager(self)
  load_internal()
  fill()

FUNCTION load_internal()
  thunderbolts.load()
  suns.load()
  levels.load()
  effects.load()
  sound_channels.load()
  ambients.load()
  base_environment.load()        # which calls load_weathers() through the base
```

**Invariants** — construction order is not load order, and both matter. Construction order
is forced by references: the ambients manager reads the effects and sound-channel managers
through this object, and the levels manager holds the weathers manager directly. Load
order is forced by containment: ambients name effects and sound channels, so those load
first; the base system's own load reaches back into this object for thunderbolt and
ambient records, so it runs last.

**Notes** — Note what *is not* ordered: nothing loads weather cycles here. The base
system's load calls `load_weathers` on the way through, which is how the editor's cycles
get read without this file naming them. **The base weather system's load sequence is
preserved exactly; the editor only substitutes what it builds, never when.** That is what
lets the editor and the game agree about the world.

## `fill` — building the grid's tree

```text
FUNCTION fill()
  property_holder = editor.create_property_holder("environment")
  weathers.fill(property_holder)
  suns.fill(property_holder)
  ambients.fill(property_holder)
  effects.fill(property_holder)
  sound_channels.fill(property_holder)
  thunderbolts.fill(property_holder)
  levels.fill()                          # builds its own root
  editor.populate_weather_list(property_holder)
```

**Contract** — assembles one property tree covering the whole weather model and hands it
to the editor's weather panel. The levels manager is the exception: it owns a separate
root, because level-to-cycle assignment is browsed in its own panel.

## `load_weathers`

```text
FUNCTION load_weathers()
  weathers.load()
  FOR EACH cycle IN weather_cycles
    REQUIRE length(cycle.frames) > 1      # "Environment in weather must >=2"
    sort cycle.frames BY exec_time
  REQUIRE weather_cycles IS NOT empty
  set_weather(first cycle by name)
```

**Invariants** — a cycle must have at least two keyframes, because a blend needs a start
and a target, and a one-keyframe cycle would blend with itself forever. Keyframes are
sorted by time of day; every consumer relies on it, including the midnight wrap in
[`engine_impl.cpp`](engine_impl.cpp.md).

**Notes** — Selecting the first cycle alphabetically on load is arbitrary but necessary:
the editor must be looking at *something* before the author picks. The selection is
restored from the author's saved session afterwards, by the editor side.

## `save`

```text
FUNCTION save()
  weathers.save()
  ambients.save()
  effects.save()
  sound_channels.save()
  thunderbolts.save()
  # suns and levels are deliberately not saved here
```

**Notes** — Two sub-managers are excluded, each for its own stated reason. Suns are not
saved because the editor exposes only part of a sun's authored record — writing the file
would drop the fields it never read; see
[`editor_environment_suns_sun.cpp`](editor_environment_suns_sun.cpp.md). Levels are not
saved here because the level-to-cycle assignment is written when the editor exits, not on
the save command.

**This is the recipe's sharpest warning about the file set: a partial reader must not
become a writer.** The weather keyframe files are safe to rewrite because the editor reads
and writes every field; the suns file is not. A rebuild that completes the sun surface may
then save it, and should.

## The three browsable name lists

```text
FUNCTION shader_ids() -> list<text>          # cached after first call
  IF cache IS NOT empty THEN RETURN cache
  open the shader library file from the game data folder
  read its third chunk
  count = read int
  REPEAT count TIMES: read a zero-terminated string INTO cache
  sort cache BY natural order
  RETURN cache

FUNCTION particle_ids() -> list<text>        # cached after first call
  IF cache IS NOT empty THEN RETURN cache
  FOR EACH group IN renderer.particle_library
    append group.identifier TO cache
  sort cache BY natural order
  RETURN cache

FUNCTION light_animator_ids() -> list<text>  # cached after first call
  IF cache IS NOT empty THEN RETURN cache
  FOR EACH animator IN light_animation_library
    append animator.name TO cache
  sort cache BY natural order
  RETURN cache
```

**Contract** — three lists of names the author picks from in the grid: material passes,
particle systems, and light animation curves. Each is built on first request and never
rebuilt. None blocks after the first call; the first call for shaders reads a file.

**Invariants** — all three are sorted with the *natural* order described in
[`editor_environment_detail.cpp`](editor_environment_detail.cpp.md), not plain text order,
so `flare10` sorts after `flare9`. Every browsable list in this module uses that ordering,
which is what makes the pickers navigable.

**Notes** — Chunk 3 of the shader library is where its material-pass names live; the
number is a format constant of that file and nothing here explains it. The shader list is
the only one read from a file — the other two are already in memory because the renderer
and the animation library loaded them for the running level.

The emptiness test doubles as the cache's validity flag, which means an empty result is
re-read on every request. Harmless: none of the three is empty in a real installation.

## The three factory hooks

```text
FUNCTION create_ambient(section) -> Ambient
  RETURN ambients.get(section)

FUNCTION thunderbolt_description(config, section) -> ThunderboltDescription
  RETURN thunderbolts.description(config, section)

FUNCTION thunderbolt_collection(section) -> ThunderboltCollection
  RETURN thunderbolts.collection(section)
```

**Contract** — the base weather system calls these while loading keyframes, expecting to
build records from configuration. Here they return the *already-built editable* records
instead, looked up by name, and the configuration argument is ignored.

**Notes** — This is the substitution in its purest form: **the base system asks for a
record and gets the one the author is editing.** It is why the sub-managers must load
before the base system does. It also means an unknown name is a programming error, not a
data error — the lookups fail hard rather than building something.

## `create_mixer` and `unload`

```text
FUNCTION create_mixer()
  REQUIRE current_env IS none
  current_env = new EditableKeyframe(self, owner = none, id = "")
  current_env.fill(collection = none)

FUNCTION unload()
  clear weather_cycles, weather_effects, modifiers, ambients
```

**Contract** — `create_mixer` builds the keyframe that holds interpolated results. It has
no owning cycle and an empty identifier, and it is registered with the grid but not with
any editable list — so it appears as a property page with no add or remove.

**Notes** — Making the interpolation target a *real editable keyframe* rather than a plain
record is the decision that lets the editor show the author the value the engine is
actually using, in the same grid, in the same units, beside the two keyframes they wrote.
Its identifier is rewritten to the current clock time on every blend, which is how the
editor reads back the time of day as text. See
[`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md).

## Teardown

Sub-managers are released in an order that is not the reverse of construction — the
thunderbolts and ambients go first, the suns last. The cycle and effect tables are cleared
before anything else so that no record outlives its manager. The grid's root is released
only if an editor is still attached.

**Notes** — Release order matters here only because the sub-managers reference each other;
in a rebuild whose element lifetimes are owned rather than manual, this whole sequence
disappears, and what survives is the constraint: **no sub-manager may outlive the tables
that point into it.**
