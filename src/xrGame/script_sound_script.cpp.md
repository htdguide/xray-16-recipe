# src/xrGame/script_sound_script.cpp

> Exports the script-owned sound handle and the emitter parameter block to the script virtual machine.

**Needs** — [`script_sound.h`](script_sound.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

This is how a script makes noise. Two types are declared to Lua: the emitter parameter
block, and the sound handle itself with its play/stop surface. Every name and every
overload here is frozen by conformance criterion 10 — the shipped scripts call all of
them.

## State

`Stateless.`

## `CScriptSound::script_register`

**Contract** — registers, once at script-engine bring-up:

- **`sound_params`** — five read/write fields on the emitter parameter block: position,
  volume, frequency, minimum distance, maximum distance. Exported as a struct so a script
  can capture a playing sound's whole parameter set and reapply it.

- **`sound_object`** — the handle. Constructible from a sound name alone, or from a name
  plus an AI-perception sound type (which decides what creatures hearing it make of it).
  It carries:
  - a nested enumeration `sound_play_type` with three values — `looped`, `s2d`
    (non-positional, mixed flat) and `s3d` (positional, the zero value). These are *flag
    bits*, not an exclusive choice: `looped` and `s2d` combine, and `s3d` being zero is
    what makes positional-and-once the default when a script passes no flags.
  - four properties — frequency, minimum distance, maximum distance, volume — each a
    read/write pair.
  - `play` in three arities (object; object and delay; object, delay and flags) and
    `play_at_pos` in three matching arities (adding an explicit position). The short forms
    exist so scripts written against the original API keep working; each fills the missing
    arguments with zero.
  - `play_no_feedback`, which starts a sound the handle does not keep a channel for — a
    fire-and-forget emission the script cannot subsequently stop or query.
  - position get and set, `stop`, `playing`, `length`, and `attach_tail`.

**Notes** — deferred stop is registered under **two** names, one of which is a misspelling
(`stop_deffered`) kept alive because shipped scripts call it. Both bind the same
operation. A rebuild must export both spellings; dropping the typo breaks shipped content.
