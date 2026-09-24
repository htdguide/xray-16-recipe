# src/editors/xrWeatherEngine/editor_environment_sound_channels_channel.cpp

> One sound channel: a pool of interchangeable sound files, how far they carry, and the two intervals that decide when the next one starts.

**Needs** — [`editor_environment_sound_channels_channel.hpp`](editor_environment_sound_channels_channel.hpp.md) · [`editor_environment_sound_channels_manager.hpp`](editor_environment_sound_channels_manager.hpp.md) · [`editor_environment_sound_channels_source.hpp`](editor_environment_sound_channels_source.hpp.md) · [`property_collection.hpp`](property_collection.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)
**Used by** — [`editor_environment_sound_channels_channel.hpp`](editor_environment_sound_channels_channel.hpp.md)
**Tier floor** — T2: it owns its source list and mirrors the engine's.

## Purpose

A sound channel is a *pool with a pacing rule*: several sound files that mean the same
thing (four different wind gusts, three different crow calls), a distance range, and two
intervals — how long to wait before starting, and how long to pause between.

## State

See [`editor_environment_sound_channels_channel.hpp`](editor_environment_sound_channels_channel.hpp.md).

## The sound-channel record on disk

```text
RECORD SoundChannelSection              # one configuration section per channel
  section name  : text
  min_distance  : real                  # metres
  max_distance  : real                  # metres
  period0       : int                   # minimum start interval, seconds
  period1       : int                   # maximum start interval, seconds
  period2       : int                   # minimum pause interval, seconds
  period3       : int                   # maximum pause interval, seconds
  sounds        : text                  # comma-separated sound paths, extension dropped
```

**Invariants** — the four periods are a pair of ranges, in the order
(start min, start max, pause min, pause max). Nothing enforces that each minimum is below
its maximum.

**Notes** — **This layout is frozen**, numbered field names and all. The names carry no
meaning; the editor's row labels are where the meaning is written down, and they are the
only documentation of it in the project — which is why the grid's descriptions are
reproduced here.

## `load` and `save`

```text
FUNCTION load(config, section_to_read_from)
  base_channel.load(config, id, section_to_read_from)     # distances and the four periods
  REQUIRE sources IS empty
  FOR EACH path IN split(config.text(id, "sounds"))
    append new Source(path) TO sources, registered with collection

FUNCTION save(config)
  write min_distance, max_distance, period0..period3
  write "sounds" = join(sources, ", ")
```

**Contract** — the base loader reads the numbers and resolves the sound list into playable
sources; this file reads the same list again as editable paths. The optional
"read from a different section" argument lets one channel inherit another's numbers — the
base loader's feature, passed through.

**Notes** — Two name lists built from one text, never reconciled — the same duplication
described in
[`editor_environment_ambients_ambient.cpp`](editor_environment_ambients_ambient.cpp.md),
and the same consequence: editing the list changes what is saved, not what is currently
playing.

Construction zeroes the distances and the four periods before loading, so a section
missing a field gets zero rather than the previous channel's value.

## `fill` — the channel's eight rows

```text
GROUP properties : id (text, filtered through the manager's naming rule)
                   minimum distance (real, metres)
                   maximum distance (real, metres)
                   period 0 — minimum start interval, seconds
                   period 1 — maximum start interval, seconds
                   period 2 — minimum pause interval, seconds
                   period 3 — maximum pause interval, seconds
                   sounds (an editable list of sound paths)
```

**Notes** — The rows keep the file's opaque names and put the meaning in the description,
rather than renaming them. That is the right call for a tool whose job is to write a frozen
format: **the author sees the field they are editing, and the explanation beside it.**
