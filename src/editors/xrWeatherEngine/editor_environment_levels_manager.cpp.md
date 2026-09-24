# src/editors/xrWeatherEngine/editor_environment_levels_manager.cpp

> Which weather cycle each level plays: read from the two level catalogues, shown as one row per level.

**Needs** — [`editor_environment_levels_manager.hpp`](editor_environment_levels_manager.hpp.md) · [`editor_environment_weathers_manager.hpp`](editor_environment_weathers_manager.hpp.md) · [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md) · [`xrCore/LocatorAPI.h`](../../xrCore/LocatorAPI.h.md)
**Used by** — [`editor_environment_levels_manager.hpp`](editor_environment_levels_manager.hpp.md)
**Tier floor** — T2: it holds two configuration files open and rewrites them.

## Purpose

A weather cycle is authored once and assigned per level. The assignment lives in the level
catalogues — the same files that list which levels exist — so editing it means editing a
file that is not part of the weather model. This manager does exactly that, and no more.

## State

See [`editor_environment_levels_manager.hpp`](editor_environment_levels_manager.hpp.md).

## `load`

```text
CONSTANT default_weather_id = "[default]"       # a level with no explicit cycle
CONSTANT level_section_id   = "levels"          # unused in the shipping code path

FUNCTION load()
  single_config = read "$game_config$/game_maps_single.ltx", keeping it open
  mp_config     = read "$game_config$/game_maps_mp.ltx",     keeping it open
  REQUIRE levels IS empty
  collect_levels(FROM single_config, index section "level_maps_single", category "single")
  collect_levels(FROM mp_config,     index section "level_maps_mp",     category "multiplayer")

FUNCTION collect_levels(config, index_section, category)
  FOR EACH key IN config.section(index_section)
    IF key IS empty THEN CONTINUE
    REQUIRE config has a section named key
    IF config.section(key) has no "weathers" entry
      levels[key] = (category, default_weather_id)
    ELSE
      levels[key] = (category, config.text(key, "weathers"))
```

**Contract** — each catalogue has an index section listing level names, and one section per
level. The weather cycle is that section's `weathers` entry, or a sentinel when absent.
Both catalogues feed one map, so a level name appearing in both is stored once.

**Invariants** — a level named in the index must have a section of its own. The two
catalogues are read into a single map sorted by level name; the category distinguishes
them afterwards.

**Notes** — **The files are held open, not closed after reading, because they will be
rewritten on exit**, and the original states the consequence plainly beside the load:
rewriting drops every comment and sorts every section. That is a destructive edit to a
file the editor barely touches — it changes one entry per level and normalises the rest —
and it is the strongest argument in this module for a configuration writer that preserves
what it did not change. A rebuild should write back in place.

The empty-key test exists because the index section's entries are bare names with no
values, and the configuration reader yields an empty trailing key for the section's own
formatting. Incidental.

The sentinel `[default]` is a string, not an absent value: a level with no explicit cycle
shows that text in its row, and picking a real cycle replaces it. Choosing it *back* is
impossible through the grid, because the sentinel is not in the cycle list — so the
assignment is one-way. A rebuild should offer an explicit "no cycle" option.

## `fill` — one row per level

```text
FUNCTION fill()
  property_holder = editor.create_property_holder("levels")
  FOR EACH (level_name, (category, cycle)) IN levels
    property_holder.add_property(
        level_name, group category,
        description = "weather for level " + level_name,
        value bound to the stored cycle,
        options pulled from weathers.cycle_ids,
        editor = combo box, free text not allowed)
  editor.populate_level_list(property_holder)
```

**Contract** — builds its own property root — not a branch of the weather tree — and hands
it to the editor's level panel. One row per level, grouped by whether the level is
single-player or multiplayer, each a combo box over the cycle names.

**Notes** — Separating this root from the weather root is the editor's information
architecture again: the weather panel is where cycles are authored, the levels panel is
where they are assigned, and the two are different tasks with different rhythms.

The options are pulled from the weathers manager at paint time, so a cycle renamed in the
weather panel changes what this panel offers — but not what an already-assigned level
stores. Same text-reference limitation as everywhere else in this module.

## `save` — declared and never written

The header declares a save operation; no definition exists anywhere in the module, and the
one call site — in [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) —
is commented out with the note that level assignments are written when the editor exits
instead.

**Notes** — So the level catalogues are written by whatever runs at exit; nothing in this
module does it, and the write-on-close configurations held open since load are the
mechanism. That is why they are held: **closing them is the save.** It is implicit,
unconditional, and happens even if the author changed nothing — which, combined with the
comment-stripping rewrite noted above, means opening the weather editor and closing it
reformats both level catalogues. A rebuild should make the save explicit and conditional.
