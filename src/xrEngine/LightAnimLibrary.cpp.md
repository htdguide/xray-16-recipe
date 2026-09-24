# src/xrEngine/LightAnimLibrary.cpp

> The library of named colour animations — the engine's answer to "make this light flicker like a campfire".

**Needs** — [`LightAnimLibrary.h`](LightAnimLibrary.h.md) · [`xrCore/FS.h`](../xrCore/FS.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`LightAnimLibrary.h`](LightAnimLibrary.h.md)
**Tier floor** — T2: a keyframe table and a lerp; the only T1 pressure is the frozen on-disk chunk layout, which a rebuild may parse field by field

## Purpose

Lights, glows, screen effects and some UI elements in the shipped game are animated by
*colour over time*: a fluorescent tube stutters, a campfire pulses orange, an anomaly
throbs. Rather than script each of those, the game data ships one file of named colour
animations and everything that wants to flicker names one. This file owns that file, its
in-memory form, and the evaluation.

It is one library and not per-level because the same animations are referenced from
several levels and from script, and because the whole set is small enough (a few dozen
curves of a few keys each) that partitioning it would cost more than it saves.

## State

```text
RECORD ColorAnimation            # CLAItem
  name        : text
  fps         : real             # authored playback rate; default 15
  frame_count : int              # total length in frames; at least 1
  keys        : map<int, int (32-bit, packed RGBA)>   # sparse: frame index -> colour

# invariant: keys is never empty for a loaded animation; key 0 always exists
#            (a default-initialized animation gets frame 0 = transparent black)
# invariant: every key frame index is <= frame_count
# invariant: colours in memory are always in RGBA order, whatever the file said

RECORD Library                   # ELightAnimLibrary
  items : list<ColorAnimation>   # unique by name

# invariant: names are unique; appending a duplicate is a programming error
```

Length in seconds is `frame_count / fps`; length in milliseconds is that floored — the
millisecond form is what script sees.

## File format

One library file under the game-data root, in the engine's recursive chunked container.

```text
CHUNK 0x0000  version : int (16-bit)          # absent in the oldest files, which means 0
CHUNK 0x0001  item list
    CHUNK <n>  one animation, n counting from 0 with no gaps
        CHUNK 0x0001  name : stringZ, fps : real, frame_count : int
        CHUNK 0x0002  key_count : int, then key_count pairs of (frame : int, colour : int)
```

**Frozen.** The version field carries exactly one meaning: **version 0 stored colours with
the blue and red channels swapped**, and the loader swaps them back while preserving alpha.
Everything written by this engine is version 1 and needs no fixup. A rebuild that skips
this fixup renders every shipped flicker in the wrong hue.

The item-list chunk is read by probing sub-chunk ids 0, 1, 2 … until one is missing, so
the numbering must be dense.

## `Load`

**Contract** — reads the library file if it exists; a missing file is not an error and
leaves the library empty, because a level can legally reference no animations. Appends to
whatever is already loaded rather than replacing, so `Unload` must precede a reload.

```text
FUNCTION load(library)
  file = open(game_data_root / "lanims.xr")
  IF file IS none
    RETURN                                  # not an error
  version = 0
  IF file HAS chunk VERSION
    version = read int (16-bit)
  list = file.open_chunk(ITEM_LIST)
  IF list IS none
    RETURN
  n = 0
  WHILE list HAS chunk n
    item = read_animation(list.chunk(n))
    IF version == 0
      FOR EACH key IN item.keys
        key.colour = swap red and blue, keep alpha
    library.items.append(item)
    n = n + 1
```

## `Save`

**Contract** — writes the whole library to a memory buffer and then to the library file in
one act; a failed write is logged, not fatal. Always stamps the current version, so saving
an old file silently upgrades it. Used only by the editors.

## `InterpolateRGB`

**Contract** — the colour at an exact frame index. Returns a key's colour verbatim when
the frame is a key; otherwise linearly interpolates between the surrounding keys in
straight RGBA space. Past the last key the last key's colour holds — animations do not
wrap here, wrapping is the caller's (see `CalculateRGB`). Before the first key cannot
happen, because key 0 always exists.

```text
FUNCTION interpolate(anim, frame) -> int (32-bit, packed)
  IF frame IS a key
    RETURN keys[frame]
  next = first key strictly after frame
  IF next does not exist
    RETURN colour of the last key
  prev = the key immediately before next
  t = (frame - prev.frame) / (next.frame - prev.frame)
  RETURN lerp(prev.colour, next.colour, t)    # component-wise, including alpha
```

**Notes** — interpolation is per-component in straight (not premultiplied, not
gamma-corrected) space. That is visibly wrong for a colour ramp and is exactly what the
shipped art was authored against, so a rebuild must reproduce it rather than improve it.

## `CalculateRGB`

**Contract** — the colour at a *time* in seconds, looping. Wraps the time into the
animation's own length and floors to a frame, then interpolates. Also reports the frame it
landed on, which callers use to detect a wrap. The animation always loops; there is no
one-shot mode.

```text
FUNCTION calculate(anim, t_seconds) -> (colour, frame)
  frame = floor( (t_seconds MOD (frame_count / fps)) * fps )
  RETURN (interpolate(anim, frame), frame)
```

## `InterpolateBGR` / `CalculateBGR`

**Contract** — the same two results with red and blue exchanged. They exist because two
consumers — parts of the renderer and the older vertex formats — want the opposite channel
order, and swapping at the call site was error-prone. A rebuild with one canonical colour
order deletes both.

## `Resize`

**Contract** — changes an animation's length in frames, keeping at least one. Growing
*moves* the key that sat on the old last frame to the new last frame, so the final colour
of the loop stays final; shrinking discards every key past the new end. This asymmetry is
authoring behaviour: stretching an animation should stretch its ending, truncating it
should simply cut.

## `InsertKey` / `DeleteKey` / `MoveKey`

**Contract** — sparse-map edits, all bounded by the current frame count. Deleting frame 0
is silently refused — the animation must always have a key at its origin, or evaluation
before the first key would be undefined.

## `PrevKeyFrame` / `NextKeyFrame`

**Contract** — navigate to the neighbouring key for the editors' timeline. Both clamp at
the ends by returning the last key rather than failing.

**Notes** — `FirstKeyFrame` as written in the header dereferences the end of a reverse
traversal, which is a defect; nothing in the engine calls it. A rebuild should return the
lowest key index, which is always 0.

## `FindItem` / `AppendItem`

**Contract** — lookup is a linear scan comparing names case-sensitively; the library is
small enough that this is never measured. `AppendItem` creates a new animation, optionally
copying an existing one as a template, and asserts the name is not already taken —
duplicate names would make lookup order-dependent.

## Script surface: `color_animator`

**Contract** — the script-visible facade over one animation. Constructed from a name and
fails hard if no such animation exists, because a mistyped name in a script would
otherwise silently produce black. Exposes the animation's length in milliseconds and a
colour at a time in seconds.

```text
CLASS color_animator
  new(name)            # fails if the name is unknown
  load(name)
  length() -> int      # milliseconds
  calculate(t_seconds) -> colour
```

**Notes** — the wrapper holds a bare reference into the library, so unloading the library
while a script holds one is unsafe. In practice the library outlives every script state.
