# src/xrSound/SoundRender_Environment.cpp

> A reverb preset, its blend, and the library of presets a level's environment geometry names.

**Needs** — [`SoundRender_Environment.h`](SoundRender_Environment.h.md) · [`SoundRender.h`](SoundRender.h.md) · [`SoundRender_EffectsA_EAX.h`](SoundRender_EffectsA_EAX.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: the preset file is a frozen chunked binary read field by field, and the value
ranges are the reverb model's, not the engine's.

## Purpose

A preset describes what a room does to sound. The engine does not model rooms; it names them, and
hands the named description to the mixer's reverb. This file holds the description, the blend that
makes crossing a boundary gradual, and the library that maps a level's per-face names to presets.

## State

```text
RECORD Environment
  version : int
  name    : text          # the key level geometry refers to it by
  kind    : int           # a preset index into the reverb model's own room catalogue

  room                  : real   # room effect level at low frequencies
  room_hf               : real   # high-frequency room level, relative to low
  room_rolloff_factor   : real   # distance rolloff applied to the room effect only
  decay_time            : real   # reverberation decay at low frequencies, seconds
  decay_hf_ratio        : real   # high- to low-frequency decay time ratio
  reflections           : real   # early reflections level, relative to the room effect
  reflections_delay     : real   # time to the first reflection
  reverb                : real   # late reverberation level, relative to the room effect
  reverb_delay          : real   # late reverb delay, relative to the first reflection
  environment_size      : real   # the modelled room's size in metres
  environment_diffusion : real   # echo density during the decay
  air_absorption_hf     : real   # level change per metre at 5 kHz
```

Invariants:

- Every field is clamped to the reverb model's own legal range after any write. The ranges belong
  to the model, not to the engine, which is why a blend that is mathematically fine can still need
  clamping: interpolating two legal presets stays legal, but accumulated rounding does not have to.
- `name` is the only identity. Level geometry names presets by string at author time; those names
  are resolved to library indices once at level load (see
  [`SoundRender_Scene.cpp`](SoundRender_Scene.cpp.md)).

## `set_default` / `set_identity`

**Contract** — `set_default` fills every field with the reverb model's own neutral defaults — a
plausible generic room. `set_identity` is that, with the room level pushed to its *minimum*, which
is the "no reverb at all" preset: outdoors, and the fallback wherever geometry answers nothing.

**Notes** — Identity is not zero and not default. It is the one preset that must be audibly
*absent*, and it is what every emitter and the listener start at.

## `lerp`

**Contract** — Linear blend of two presets into this one, by a factor in [0,1], followed by a clamp.
The `kind` and `name` are not blended — a blend has no identity, only parameters.

**Notes** — Every field blends independently and linearly, including the ones that are logarithmic
levels and the ones that are times. That is not physically principled — crossfading two decay times
is not the decay time of a crossfaded room — but it is cheap, monotone and what the level designers
tuned against. The blend factor supplied by both callers is the frame delta, which makes the
approach frame-rate dependent; see
[`SoundRender_Core.cpp`](SoundRender_Core.cpp.md).

## `load` / `save`

**Contract** — Read and write one preset from a chunk of the preset library file. Refuses versions
below 3. Version 4 appends the room-kind word; earlier files leave it at whatever the defaults set.
Field order is the record order above and is **frozen** — the shipped preset file is game data.

```text
FUNCTION load(reader) -> bool
  version ← reader.int
  IF version < 3 THEN RETURN false
  name ← reader.string
  read the twelve real fields in declaration order
  IF version ≥ 4 THEN kind ← reader.int
  RETURN true
```

## `SoundEnvironmentLibrary`

**Contract** — The set of presets a level's environment geometry may name. Loaded once from a
single file in the game data; the file is a chunked container with one preset per chunk, and a
chunk that fails to load is skipped rather than failing the load. Presets are addressed by index
(what geometry stores) or by name (what an author writes), and name lookup is case-insensitive.
The editor additionally appends, clones and removes entries, which is why the library is mutable at
all — in the game it is read-only after load.

**Invariants** — Indices are positions in the library, so the library must not be reordered between
loading the presets and resolving a level's geometry against them. `GetID` returning "not found" is
a fatal condition at level load: a level naming a preset the library does not have cannot be made
to sound right, and silently substituting identity would hide an authoring error.

**Notes** — Reserving space for 256 presets on load is a sizing hint, not a limit.

## Notes

Every field name, default and legal range in this file comes from one specific vendor reverb model,
and the values are meaningless outside it. A rebuild on a different reverb has two honest options:
map these twelve parameters onto the new model's (most reverbs expose recognisable equivalents of
decay time, HF ratio, early/late levels and their delays), or treat `name` as the real payload and
author a fresh preset per name. The second is more work and sounds better; the first loads the
shipped preset file unchanged, which is what the conformance criteria ask for.
