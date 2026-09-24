# src/xrGame/script_sound_info.h

> The record a creature's sound-perception callback hands to a script: who made the noise, where, how loud, when, and whether it counts as dangerous.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_game_object_script2.cpp`](script_game_object_script2.cpp.md) · [`script_game_object_use2.cpp`](script_game_object_use2.cpp.md) · [`script_sound_info_script.cpp`](script_sound_info_script.cpp.md)
**Tier floor** — T3: a plain value record with one mutator

## Purpose

*Feel*'s sound channel is event-driven: when a sound event reaches an entity the engine
fills one of these and passes it into the script layer. The record is a separate type
rather than five callback arguments because a script stores it — a creature remembers the
last noise it heard across frames — and because the binding layer exports it as a mutable
object scripts can read field by field.

The type is declared entirely in this header; only its script registration lives
elsewhere ([`script_sound_info_script.cpp`](script_sound_info_script.cpp.md)).

## State

```text
RECORD ScriptSoundInfo
  who       : optional<GameObject>   # the emitter's script facade; none when unknown
  position  : vector3                # where the sound was emitted, not where it was heard
  power     : real                   # perceived loudness after distance attenuation
  time      : int                    # global clock at which the event arrived
  dangerous : int                    # 0 or 1 — the AI-perception danger flag, widened
                                     # from a boolean so the script side sees a number
```

**Invariants** — a default-constructed record is the *no sound heard* marker: emitter
none, time zero, power zero, position at the origin, not dangerous. Readers test `who`
or `time`, not `power`, to decide whether anything was heard at all.

## `ScriptSoundInfo`

**Contract** — default construction yields the empty marker above. The single mutator
takes emitter, danger flag, position, power and timestamp and overwrites every field
together, so the record is never half-updated: a script that samples it mid-frame sees
either the previous event whole or the new one whole.

**Notes** — the danger flag is stored widened to an integer because the exported field is
read *and written* from script, and the binding layer's boolean conversion was not worth
the special case. A rebuild should keep the exported name (`danger`) and accept a number.
