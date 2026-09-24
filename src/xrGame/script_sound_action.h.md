# src/xrGame/script_sound_action.h

> Declares the sound channel of a scripted action: a sound to play from a bone or a place, with monster and trader variants folded into the same record.

**Needs** — [`script_abstract_action.h`](script_abstract_action.h.md) · [`script_sound.h`](script_sound.h.md) · [`ai_sounds.h`](../xrServerEntities/ai_sounds.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md) · [`ai/monsters/monster_sound_defs.h`](ai/monsters/monster_sound_defs.h.md)
**Used by** — [`script_entity_action.h`](script_entity_action.h.md) · [`script_sound_action.cpp`](script_sound_action.cpp.md) · [`script_sound_action_inline.h`](script_sound_action_inline.h.md) · [`script_sound_action_script.cpp`](script_sound_action_script.cpp.md)
**Tier floor** — T2: a record with three mutually exclusive modes

## Purpose

The channel of a [scripted action](script_entity_action.h.md) that makes noise. It carries
three unrelated orders in one record, and which one is live is decided by the constructor
and recorded nowhere:

1. **a named sound**, attached to a bone or standing at a position — the general form;
2. **a monster sound kind**, which asks the creature's own sound bank for one of its
   authored vocalizations rather than naming a file;
3. **a trader sound**, which is the general form plus a head-animation to wear while it
   plays.

Unlike the [particle channel](script_particle_action.h.md), this one owns no live
resource — it carries only the sound's *name*, and the entity playing the action creates
the emitter. That is what makes the channel cheap to build and copy.

## State

```text
RECORD ScriptSoundAction EXTENDS ActionChannel   # the base supplies `completed`
  sound_name          : text
  bone_name           : text
  goal_type           : enum { attached, positioned, none }
  looped              : bool
  started             : bool          # has the entity handed it to the audio seam yet
  position            : vector        # offset from the bone, or world position
  angles              : vector        # orientation, for a directional emitter
  sound_kind          : enum          # the AI perception attribute; see the export
  monster_sound       : enum          # mode 2: which of the creature's own vocalizations
  monster_sound_delay : int           # −1 means "the creature's own default delay"
  head_anim           : enum { none, normal, angry, glad, kind }   # mode 3
```

**Invariants**

- `goal_type` follows the same last-writer-wins rule as the other channels: naming a sound
  tags it *attached*, setting a position tags it *positioned*. The four general
  constructors get their intended tag purely from the order of their calls — see
  [`script_sound_action_inline.h`](script_sound_action_inline.h.md).
- `monster_sound` defaults to the marker value meaning "no monster sound", so a channel
  built in the general form does not accidentally ask a creature to vocalize.
- `head_anim` defaults to *none*, so a channel built in the general or monster form does
  not disturb a trader's expression.
- `started` is cleared by every setter and by `initialize`, so a channel re-entered from
  the top of the queue plays again rather than being skipped as already-handled. This is
  the one channel whose `initialize` does something.

## Exported units

- construct — inert.
- construct from (sound name | script sound, bone name | position, placement, looped,
  sound kind) — four general forms.
- construct from (monster sound kind) and (monster sound kind, delay).
- construct from (sound name, bone name, head animation) — the trader form.
- destroy — releases nothing; the channel owns no emitter.
- `set_sound(name)` and `set_sound(script sound)` — name it directly, or copy the name out
  of a script-owned sound handle.
- `set_position`, `set_bone`, `set_angles`, `set_sound_type`.
- `initialize` — clears the started flag.

**Notes**

`set_sound` taking a script-owned sound handle copies **only its name**, not its emitter.
Two things follow: the handle may be destroyed immediately afterwards, and any tuning
already applied to that handle — volume, range, frequency — is lost. A script that wants a
tuned sound in an action must retune it after the entity starts it, which is awkward and is
why most content names the sound directly.
