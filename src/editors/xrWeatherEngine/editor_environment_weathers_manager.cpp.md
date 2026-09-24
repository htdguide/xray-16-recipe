# src/editors/xrWeatherEngine/editor_environment_weathers_manager.cpp

> Owns every weather cycle: discovers them as files, exposes them to the timeline, and routes clipboard and reload commands to the right one.

**Needs** — [`editor_environment_weathers_manager.hpp`](editor_environment_weathers_manager.hpp.md) · [`editor_environment_weathers_weather.hpp`](editor_environment_weathers_weather.hpp.md) · [`editor_environment_weathers_time.hpp`](editor_environment_weathers_time.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`editor_environment_detail.hpp`](editor_environment_detail.hpp.md) · [`property_collection.hpp`](property_collection.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`xrCore/LocatorAPI.h`](../../xrCore/LocatorAPI.h.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md)
**Used by** — [`editor_environment_weathers_manager.hpp`](editor_environment_weathers_manager.hpp.md)
**Tier floor** — T2: it enumerates a virtual filesystem folder and routes text buffers.

## Purpose

The top of the document model. A weather cycle is a file; the set of cycles is the set of
files in one folder; and this manager is what turns that convention into a list the author
can add to, rename, browse and save.

## State

See [`editor_environment_weathers_manager.hpp`](editor_environment_weathers_manager.hpp.md).

## `load` — cycles are files

```text
FUNCTION load()
  FOR EACH name IN filesystem.list("$game_weathers$")
    IF length(name) <= 4 THEN CONTINUE
    IF name DOES NOT END WITH ".ltx" THEN CONTINUE
    id = name WITHOUT its last four characters
    cycle = new Weather(environment, id)
    cycle.load()
    cycle.register_with(collection)
    append cycle TO cycles
```

**Contract** — discovers every weather cycle by listing one folder and keeping the
configuration files. The cycle's identifier is its filename without the extension, so
**a cycle's name and its file name are the same fact**, and renaming a cycle in the grid
renames the file it will be saved to.

**Notes** — The extension test is written out character by character in the original
rather than as a suffix comparison; that is incidental. What is not incidental is that the
test is exact and lowercase: a cycle file named with an uppercase extension is invisible
to the editor on a case-sensitive filesystem, and the game would still load it. A rebuild
should compare case-insensitively.

Nothing here reads a manifest. **The weather folder is the manifest** — which is why an
author creates a cycle by adding a grid row rather than by editing an index, and why a
cycle deleted from the grid still has its file on disk until the folder is tidied by hand.

## `save`

```text
FUNCTION save()
  FOR EACH cycle IN cycles
    cycle.save()
```

**Notes** — Every cycle is rewritten, not only the changed ones. The editor keeps no
per-cycle dirty flag, so the cheap correct choice is to write them all; the cost is that
every cycle file's comments and section order are normalised on any save. See the warning
in [`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) about partial
readers becoming writers — here the reader is complete, so rewriting is safe.

## `fill` — the timeline's four data sources

```text
FUNCTION fill(holder : PropertyHolder)
  holder.add_property("weathers", collection)
  editor.bind_weather_editor(cycle_names, cycle_count, frame_names_of, frame_count_of)
```

**Contract** — adds the cycle list to the grid, then installs four callables the editor's
timeline pulls from whenever it repaints: all cycle names, their count, the keyframe names
of a named cycle, and that cycle's keyframe count.

**Notes** — Pull, not push; see [`Include/editor/ide.hpp`](../../Include/editor/ide.hpp.md)
for why that direction is the right one. The consequence lands here: no notification
protocol exists, so adding or renaming a keyframe needs no signal.

## `cycle_ids` — the cached, sorted name list

```text
FUNCTION cycle_ids() -> list<text>
  IF NOT changed THEN RETURN cache
  changed = false
  rebuild cache FROM cycles
  sort cache BY natural order
  RETURN cache
```

**Contract** — the sorted cycle names. Rebuilt only when the collection reports a
mutation; the flag is the one it was constructed with, so adding, removing or renaming a
cycle through the grid invalidates it automatically.

**Notes** — Every "identifier list" in this module follows this exact pattern: a cache, a
flag the editable list sets, and a natural-order sort. It is the module's one concession
to the grid's habit of asking for the whole list on every repaint.

## `frame_names_of` and `frame_count_of` — the sequencing defect

```text
FUNCTION frame_names_of(cycle_id : text) -> list<text>
  discard frame_ids
  cycle = cycles WHERE id == cycle_id, ELSE RETURN none
  rebuild frame_ids FROM cycle.frames      # in stored order, NOT sorted
  RETURN frame_ids

FUNCTION frame_count_of(cycle_id : text) -> int
  cycle = cycles WHERE id == cycle_id, ELSE RETURN 0
  RETURN length(frame_ids)                 # the list the PREVIOUS call left behind
```

**Contract** — the keyframe names of a named cycle, and how many there are.

**Invariants** — **the count call is only correct if the names call ran first, for the
same cycle.** The original marks this in place as a dangerous scheme that depends on the
call sequence, and it is: the count reads a scratch list the names call filled, and does
not rebuild it. The editor's timeline happens to call them in that order, so it works.

**Notes** — This is the one place in the module where a rebuild should simply not copy the
original. Return the names and the count together, or make the count recompute — either
removes an invariant that no caller can see and no test would catch.

The keyframe names are returned in the cycle's stored order, which is time-of-day order,
*not* natural-text order — unlike every other list here. That is correct and deliberate:
the timeline's keyframe selector is a sequence along a day, not an index.

## `unique_id` — renaming a cycle

```text
FUNCTION unique_id(id : text) -> text
  IF no cycle carries id THEN RETURN id
  RETURN generate_unique_id(prefix = id)   # id + "0", id + "1", ...
```

**Contract** — accepts a proposed name unchanged when it is free, and otherwise appends
the first free decimal counter. Renaming never fails and never collides.

## Clipboard: `save_current_blend`

```text
FUNCTION save_current_blend(out buffer, buffer_size) -> ok : bool
  temp = empty configuration
  environment.current_env.save(INTO temp)          # the interpolated keyframe
  rendered = render temp AS configuration text
  IF length(rendered) > buffer_size THEN RETURN false
  append a zero byte TO rendered
  copy rendered INTO buffer
  RETURN true
```

**Contract** — serialises the *interpolated* keyframe — the value the engine is using at
this instant — into the caller's buffer, in the same configuration text the cycle files
use. Fails rather than truncating.

**Notes** — Copying the interpolated result, not the selected keyframe, is the decision
that makes the "create a keyframe from here" command work: an author scrubs to a moment
that looks right and captures exactly what they are seeing. Its section name is the
current clock time, which is why the new keyframe lands at that time.

The zero byte is appended *after* the size check and the copy length is measured before
it, which means the terminator is written into the buffer but not counted — a subtlety a
rebuild avoids by treating the payload as text with a known length rather than as a
terminated buffer.

## Clipboard and reload: routing by the current cycle

```text
FUNCTION paste_current_time_frame(buffer, size) -> ok : bool
  IF environment.current[0] IS none THEN RETURN false
  cycle = cycles WHERE id == environment.current_weather_name, ELSE RETURN false
  RETURN cycle.paste_time_frame(environment.current[0].identifier, buffer, size)

# paste_target_time_frame is identical against environment.current[1]
# add_time_frame routes the same way but names no keyframe
# reload_current_time_frame / reload_target_time_frame route the same way and return nothing
```

**Contract** — every keyframe-level command finds the cycle being edited by name, then
addresses a keyframe within it by identifier. A command with no current blend does
nothing and reports failure.

**Notes** — Routing by name on every command, rather than caching the cycle being edited,
is what keeps this manager correct across a reload: the cycle objects are destroyed and
rebuilt, and a cached reference would dangle. **The identity of the cycle being edited is
its name, held by the engine, not a reference held here.**

## `reload_current_weather` and `reload`

```text
FUNCTION reload_current_weather()
  cycle = cycles WHERE id == environment.current_weather_name
  cycle.reload()                 # discard its keyframes, re-read its file

FUNCTION reload()
  release every cycle
  load()                         # re-discover the folder
```

**Contract** — the two coarse reload granularities. Both leave the current blend pointing
at destroyed keyframes, which is why the caller in
[`engine_impl.cpp`](engine_impl.cpp.md) clears and re-selects it immediately afterwards.

**Notes** — The full reload re-lists the folder, so a cycle file added outside the editor
appears. That is the only way to bring in an externally-created cycle: there is no watch.

## Element factory: creating a cycle

```text
FUNCTION create() -> PropertyHolder          # the grid's "add" command
  cycle = new Weather(environment, generate_unique_id("weather_unique_id_"))
  cycle.register_with(this collection)
  RETURN cycle.property_holder()

FUNCTION display_name(index) -> text
  RETURN cycles[index].id
```

**Notes** — A new cycle is born named `weather_unique_id_0` and empty, and an author is
expected to rename it. It has no keyframes, which means it violates the "at least two
keyframes" invariant that
[`editor_environment_manager.cpp`](editor_environment_manager.cpp.md) enforces on load —
so a cycle created and saved without keyframes will refuse to load next time. A rebuild
should seed a new cycle with two keyframes, or enforce the invariant on save.
