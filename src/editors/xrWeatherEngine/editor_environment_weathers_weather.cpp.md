# src/editors/xrWeatherEngine/editor_environment_weathers_weather.cpp

> One weather cycle: its file, its ordered keyframes, and the rule that a keyframe's name is the time of day it takes effect.

**Needs** — [`editor_environment_weathers_weather.hpp`](editor_environment_weathers_weather.hpp.md) · [`editor_environment_weathers_time.hpp`](editor_environment_weathers_time.hpp.md) · [`editor_environment_weathers_manager.hpp`](editor_environment_weathers_manager.hpp.md) · [`editor_environment_manager.hpp`](editor_environment_manager.hpp.md) · [`property_collection.hpp`](property_collection.hpp.md) · [`ide.hpp`](ide.hpp.md) · [`xrCore/xr_ini.h`](../../xrCore/xr_ini.h.md) · [`xrCore/LocatorAPI.h`](../../xrCore/LocatorAPI.h.md)
**Used by** — [`editor_environment_weathers_weather.hpp`](editor_environment_weathers_weather.hpp.md)
**Tier floor** — T2: it reads and writes one configuration file and keeps two lists in step.

## Purpose

A cycle is the unit of authoring: one file, one name, one day's worth of keyframes. This
file holds the two decisions that make the format work — **a keyframe is a configuration
section and its section name is the time of day**, and **a cycle's keyframes are always
in time order, which is also text order.**

## State

See [`editor_environment_weathers_weather.hpp`](editor_environment_weathers_weather.hpp.md).

## `load` and `save`

```text
FUNCTION load()
  path = filesystem.resolve("$game_weathers$", id) + ".ltx"
  config = read configuration AT path
  clear environment.cycles[id]
  FOR EACH section IN config
    frame = new Time(environment, owner = self, id = section.name)
    frame.load(FROM config)
    frame.register_with(collection)
    append frame TO frames
    append frame TO environment.cycles[id]

FUNCTION save()
  path = filesystem.resolve("$game_weathers$", id) + ".ltx"
  config = new configuration AT path, write-on-close
  FOR EACH frame IN frames
    frame.save(INTO config)
```

**Contract** — reading builds one keyframe per section, in the order the file lists them;
writing emits one section per keyframe. Both derive the path from the cycle's name.

**Invariants** — the cycle's keyframes and its entry in the environment's cycle table are
the *same objects*, appended in the same order. That aliasing is what lets the engine
interpolate the author's live edits with no synchronisation step, and it is why every
mutation below touches both lists.

**Notes** — The file is not sorted on load. Sorting happens once, above, in
[`editor_environment_manager.cpp`](editor_environment_manager.cpp.md), after every cycle
is read. A rebuild can sort here instead and delete that pass; the original's split is
historical.

Saving creates the configuration in a mode that writes on close rather than appending, so
a cycle file is fully replaced. Comments and original section order do not survive. That
is acceptable here because the editor reads and writes every field of a keyframe; it is
not acceptable for the suns file, which is why that one is never written — see
[`editor_environment_manager.cpp`](editor_environment_manager.cpp.md).

## `fill` — the cycle's two rows

```text
FUNCTION fill(collection : PropertyCollection)
  property_holder = editor.create_property_holder(id, collection, owner = self)
  property_holder.add_property("id", bound to id_getter/id_setter)
  property_holder.add_property("times", this cycle's keyframe collection)
```

**Contract** — a cycle shows as two rows: its name, and its keyframe list.

## Renaming a cycle

```text
FUNCTION set_id(value : text)
  IF value == id THEN RETURN
  id = weathers_manager.unique_id(value)
```

**Notes** — The proposed name is filtered through the manager's uniqueness rule, so a
collision silently becomes `name0`. The author sees the corrected name in the grid on the
next refresh. Renaming does not rename the file — the new name takes effect at the next
save, and the old file remains.

## The keyframe naming rule

This is the load-bearing part of the file. A keyframe's identifier is a clock time, and
the editor must be able to invent a new one that is valid, free, and near where the author
was working.

```text
FUNCTION valid_id(id : text) -> bool
  # exactly the shape DD:DD:DD, digits and two colons, eight characters
  RETURN id MATCHES two digits, ":", two digits, ":", two digits

FUNCTION unique_id(current : text, proposed : text) -> text
  IF NOT valid_id(proposed) THEN RETURN current      # reject, keep the old name
  IF no keyframe carries proposed THEN RETURN proposed
  RETURN generate_unique_id(FROM proposed)

FUNCTION generate_unique_id(start : text) -> text
  REQUIRE valid_id(start)
  (h, m, s) = parse start
  # try later hours first, then later minutes, then a full scan
  FOR i FROM h+1 TO 23
    IF "ii:mm:ss" is free THEN RETURN it
  FOR i FROM m+1 TO 59
    IF "hh:ii:ss" is free THEN RETURN it
  FOR each (hour, minute, second) FROM (h, m, s+1) UPWARD TO (23,59,59)
    IF it is free THEN RETURN it
  RETURN "can not generate weather id"

FUNCTION generate_unique_id() -> text                # for a brand-new keyframe
  IF frames IS empty THEN RETURN "00:00:00"
  RETURN generate_unique_id(FROM last frame's id)
```

**Contract** — an invalid proposed name is rejected outright and the keyframe keeps the
name it had; there is no error channel, the grid simply shows the old value again. A valid
but taken name is nudged forward in time to the first free slot.

**Invariants** — the shape is fixed at eight characters with two-digit fields. Hours are
not range-checked on rename — `99:99:99` passes `valid_id` — which means the editor will
accept a keyframe time that is not a time. Loading such a cycle later parses it into a
time past the end of the day, and the keyframe never takes effect.

**Notes** — Three decisions here deserve to survive.

**A keyframe's identity is its time.** That is why renaming is a move along the day rather
than a relabel, why the list is sorted by name, and why pasting a keyframe into a
different cycle lands it at the same moment. A rebuild that gives keyframes synthetic
identifiers and a separate time field will find itself re-deriving all three behaviours.

**The search widens in three stages** — same minute and second at a later hour, then same
second at a later minute, then everything — because an author duplicating a keyframe
almost always wants it *near* the original, and stepping by an hour keeps the clean
minute-and-second the author chose. A naive "next free second" would turn `06:00:00` into
`06:00:01` and silently produce keyframes a second apart, which is invisible on the
timeline.

**The fallback is a sentence, not a failure.** With the day full — 86,400 keyframes — the
generator returns the text `can not generate weather id`, which is not a valid identifier
and will be rejected by the next rename. A rebuild should return an explicit "no free
slot" result instead; the condition is unreachable in practice but the handling is wrong.

The brand-new-keyframe case starts from the *last* keyframe rather than the current one,
so adding keyframes repeatedly walks forward through the day; an empty cycle starts at
midnight.

## `save_time_frame`

```text
FUNCTION save_time_frame(frame_id, out buffer, buffer_size) -> ok : bool
  frame = frames WHERE id == frame_id, ELSE RETURN false
  temp = empty configuration
  frame.save(INTO temp)
  rendered = render temp AS configuration text
  IF length(rendered) > buffer_size THEN RETURN false
  copy rendered INTO buffer
  RETURN true
```

**Notes** — Serialising one keyframe as a one-section configuration document is what makes
the clipboard interoperable: the text an author copies is the exact text the file would
contain, so it can be pasted into a text editor, into another cycle, or into another
running copy of the editor. See
[`Include/editor/engine.hpp`](../../Include/editor/engine.hpp.md).

## `paste_time_frame` — overwrite in place

```text
FUNCTION paste_time_frame(frame_id, buffer, size) -> ok : bool
  temp = parse buffer AS configuration
  IF temp has no sections THEN RETURN false
  frame = frames WHERE id == frame_id, ELSE RETURN false
  frame.load_from(section = temp.first_section_name, config = temp, keep_id = frame.id)
  RETURN true
```

**Contract** — replaces a keyframe's fields from a pasted block **while keeping its own
identifier**, so pasting into `06:00:00` gives that keyframe the pasted appearance without
moving it in the day. Only the first section of the pasted text is read; extra sections
are ignored.

**Notes** — Keeping the target's identifier is the whole reason `load_from` exists as a
separate operation from `load` — see
[`editor_environment_weathers_time.cpp`](editor_environment_weathers_time.cpp.md), which
has to borrow the source section's name to read the fields and then put its own back.
**Paste changes appearance, never timing.** The command that changes timing is `add`.

## `add_time_frame` — insert a new keyframe in order

```text
FUNCTION add_time_frame(buffer, size) -> ok : bool
  temp = parse buffer AS configuration
  IF temp has no sections THEN RETURN false
  section = temp.first_section_name                 # a clock time
  IF any frame carries section THEN RETURN false    # refuse to duplicate a moment
  frame = new Time(environment, owner = self, id = section)
  frame.load(FROM temp)
  frame.register_with(collection)
  index = first position WHERE frames[position].id >= section     # binary search
  frames.insert_at(index, frame)
  environment.cycles[id].insert_at(index, frame)    # keep the aliased list in step
  RETURN true
```

**Contract** — creates a keyframe at the time named by the pasted block's section, and
inserts it so the list stays sorted. Refuses if a keyframe already occupies that moment.

**Invariants** — the insertion position is computed once and applied to **both** lists, so
the cycle's keyframes and the environment's cycle table never diverge. This is the
invariant that a rebuild most easily loses, because the two lists look like a redundancy
until an insert happens at a position other than the end.

**Notes** — The sorted insert is a binary search over the keyframe identifiers, comparing
them as text — sound only because the `HH:MM:SS` shape makes text order and time order the
same, with the two-digit padding doing the work. Drop the padding and this breaks silently.

Refusing a duplicate moment rather than merging is right: two keyframes at the same
instant make the blend interval zero-length, which
[`engine_impl.cpp`](engine_impl.cpp.md) has to dodge for a different reason already.

## `reload_time_frame` and `reload`

```text
FUNCTION reload_time_frame(frame_id)
  config = read this cycle's file
  frame = frames WHERE id == frame_id, ELSE RETURN
  IF config has no section named frame.id THEN RETURN     # never saved; leave as is
  frame.load_from(frame.id, config, keep_id = frame.id)

FUNCTION reload()
  release every frame
  load()
```

**Contract** — the keyframe-level reload re-reads one section and is the editor's undo for
a single keyframe. A keyframe the author added but has not saved has no section in the
file, and is left untouched rather than emptied.

**Notes** — Together with the cycle-level and whole-model reloads above it, this is the
editor's entire undo mechanism: re-read from disk at one of four granularities. The cost
is that undo is only as fine as the last save, and the benefit is that it cannot get out
of step with the file — which, for a tool whose output must be read by the engine, is the
trade the original chose deliberately.

## Element factory: creating a keyframe

```text
FUNCTION create() -> PropertyHolder          # the grid's "add" command
  frame = new Time(environment, owner = self, id = generate_unique_id())
  frame.register_with(this collection)
  RETURN frame.property_holder()

FUNCTION display_name(index) -> text
  RETURN frames[index].id                    # the time of day
```

**Notes** — A keyframe created this way is appended to the collection wherever the grid
puts it, *not* inserted in sorted position, and it is not added to the environment's cycle
table at all. So a keyframe added from the grid is visible and editable but does not
affect the running weather until the cycle is saved and reloaded — unlike one added from
the clipboard, which does both. That asymmetry is real and a rebuild should remove it by
routing both through the same insert.
