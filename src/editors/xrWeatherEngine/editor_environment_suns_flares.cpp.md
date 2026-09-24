# src/editors/xrWeatherEngine/editor_environment_suns_flares.cpp

> A sun's flare series, decoded from four parallel lists into one editable list — in code nothing currently reaches.

**Needs** — [`editor_environment_suns_flares.hpp`](editor_environment_suns_flares.hpp.md) · [`editor_environment_suns_flare.hpp`](editor_environment_suns_flare.hpp.md) · [`editor_environment_suns_manager.hpp`](editor_environment_suns_manager.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`property_collection.hpp`](property_collection.hpp.md) · [`Include/editor/property_holder_base.hpp`](../../Include/editor/property_holder_base.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)
**Used by** — [`editor_environment_suns_flares.hpp`](editor_environment_suns_flares.hpp.md)
**Tier floor** — T2: it owns its flare list.

## Purpose

Unreachable code with a decision worth keeping. A sun's flare series is stored as **four
comma-separated lists read in parallel** — opacities, positions, radii, textures — and
this file is the only place in the project that decodes that encoding into records an
author can manipulate one at a time.

## State

See [`editor_environment_suns_flares.hpp`](editor_environment_suns_flares.hpp.md).

## The flare series on disk

```text
RECORD FlaresInASunSection            # keys inside a sun's own section
  flares          : bool    # default true
  flare_shader    : text    # default "effects/flare"
  flare_opacity   : text    # comma-separated reals, default six values
  flare_position  : text    # comma-separated reals, default six values
  flare_radius    : text    # comma-separated reals, default six values
  flare_textures  : text    # comma-separated paths, default six values
```

**Invariants** — the four lists are read positionally: entry *i* of each belongs to flare
*i*. They are *expected* to be the same length; the code does not require it.

**Notes** — **The shipping defaults are the specification of what a sun flare looks like**,
and they are written down nowhere else, so they are reproduced:

```text
opacity   0.340, 0.260, 0.500, 0.420, 0.260, 0.260
position  1.300, 1.000, 0.500, -0.300, -0.600, -1.000
radius    0.080, 0.120, 0.040, 0.080, 0.120, 0.300
textures  fx_flare1, fx_flare2, fx_flare2, fx_flare2, fx_flare3, fx_flare1
```

Position is a parameter along the line from the sun through the screen centre: values above
one sit beyond the sun, values below zero on the far side of centre. The six defaults walk
that line from `1.3` down to `-1.0`, which is why the series reads as a streak across the
frame rather than a cluster. Radius grows toward the far end. A rebuild that changes the
defaults changes how every unmodified sun looks.

## `load`

```text
FUNCTION load(config, section)
  use    = config.bool_or(section, "flares", true)
  shader = config.text_or(section, "flare_shader", "effects/flare")
  IF NOT use THEN RETURN              # a disabled series is not even decoded

  read the four lists, each with its shipping default
  counts = (count of each list)
  n = min(counts)
  IF min(counts) != max(counts)
    report: "flare count for sun [name] is setup incorrectly, only n flares are correct"

  FOR i FROM 0 TO n-1
    flare = new Flare(opacity[i], position[i], radius[i], texture[i])
    register flare WITH collection
```

**Contract** — decodes as many flares as the *shortest* list supports, reports the
mismatch, and carries on. Never fails.

**Invariants** — a disabled series has no flares at all, so switching it back on in the
grid would show an empty list rather than the authored one. That is a real bug in
unreachable code, and a rebuild should decode regardless of the switch.

**Notes** — Truncating to the shortest list and warning is the right failure mode for a
content tool: a hand-edited file with a missing radius loses one flare and says so, rather
than refusing to open the sun. The scratch buffer the original sizes from the longest
list is incidental.

## `fill`

```text
FUNCTION fill(manager, holder, collection)
  holder.add_property("use", group "flares", bound to use)
  holder.add_property("shader", group "flares", bound to shader,
                      options = manager.environment.shader_ids, tree picker, no free text)
  holder.add_property("flares", group "flares", the flare collection)
```

## `save` — declared and never written

No definition exists. Encoding four parallel lists back out of a list of records is the
missing half, and it is the reason the suns file cannot be saved; see
[`editor_environment_suns_manager.cpp`](editor_environment_suns_manager.cpp.md).

## Element factory

```text
FUNCTION create() -> PropertyHolder       # the grid's "add" command
  flare = new Flare()                     # all fields zero
  register flare WITH this collection
  RETURN flare.property_holder()

FUNCTION display_name(index) -> text
  RETURN "flare [" + flares[index].position + "]"
```

**Notes** — Labelling a flare by its position rather than by an index is a small, good
decision: the list is ordered along a line and the label says where on the line each entry
sits, so reordering is visible in the label.
