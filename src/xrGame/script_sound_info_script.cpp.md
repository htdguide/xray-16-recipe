# src/xrGame/script_sound_info_script.cpp

> Exports the heard-sound record to the script virtual machine as `SoundInfo`.

**Needs** — [`script_sound_info.h`](script_sound_info.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

One registration entry point. It exists as its own file only because the script bindings
are compiled against the script-engine precompiled header and the rest of the game is
not; a rebuild with a uniform build has no reason to keep the split.

## State

`Stateless.`

## `CScriptSoundInfo::script_register`

**Contract** — registers the class under the script name `SoundInfo` with five read/write
fields: `who`, `danger`, `position`, `power`, `time`. No constructor is exported — scripts
only ever receive an instance the engine filled, never build one. The names are frozen by
conformance criterion 10.
