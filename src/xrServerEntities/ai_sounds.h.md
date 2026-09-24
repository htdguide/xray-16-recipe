# src/xrServerEntities/ai_sounds.h

> The taxonomy of sound *as the AI hears it*: a 32-bit bitfield naming who made a noise and what kind of noise it was, carried on every sound the engine plays.

**Needs** — [`xrCore/xr_token.h`](../xrCore/xr_token.h.md) · [Data: sounds](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`CarWeapon.cpp`](../xrGame/CarWeapon.cpp.md) · [`CustomDetector.h`](../xrGame/CustomDetector.h.md) · [`Explosive.h`](../xrGame/Explosive.h.md) · [`WeaponMagazined.h`](../xrGame/WeaponMagazined.h.md) · [`monster_sound_memory.cpp`](../xrGame/ai/monsters/monster_sound_memory.cpp.md) · [`monster_sound_memory.h`](../xrGame/ai/monsters/monster_sound_memory.h.md) · [`ai_rat_feel.cpp`](../xrGame/ai/monsters/rats/ai_rat_feel.cpp.md) · [`ai_rat_impl.h`](../xrGame/ai/monsters/rats/ai_rat_impl.h.md) · [`ai_sounds.cpp`](../xrGame/ai_sounds.cpp.md) · [`memory_space.h`](../xrGame/memory_space.h.md) · [`script_sound.h`](../xrGame/script_sound.h.md) · [`script_sound_action.h`](../xrGame/script_sound_action.h.md) · [`script_sound_action_script.cpp`](../xrGame/script_sound_action_script.cpp.md) · [`sound_player.h`](../xrGame/sound_player.h.md) · _and 1 more_
**Tier floor** — T2: a bitfield whose values reach configuration and scripts.

## Purpose

Perception in this engine is event-driven: a sound is *delivered* to every creature that
could hear it, together with an attribute word saying what it was. This file is that word.
It is the reason the sound format has an engine-specific sidecar at all — a Vorbis file
cannot say "this is a weapon firing" and the AI must know.

The taxonomy is orthogonal by construction: the high bits name a **source category** and the
low bits name an **event kind**, and a complete attribute is one of each combined. A
creature's reaction table is written against whichever half it cares about — "any weapon
noise" or "anything at all that is a footstep".

## State

```text
ENUM SoundAttribute : int (32-bit bitfield)
  none = 0

  # source category — the top five bits
  weapon    = bit 31
  item      = bit 30
  monster   = bit 29
  anomaly   = bit 28
  world     = bit 27

  # event kind — the next band
  picking_up = bit 26      dropping = bit 25      hiding   = bit 24
  taking     = bit 23      using    = bit 22
  shooting   = bit 21      empty_click = bit 20   bullet_hit = bit 19
  recharging = bit 18
  dying      = bit 17      injuring = bit 16      step     = bit 15
  talking    = bit 14      attacking = bit 13     eating   = bit 12
  idle       = bit 11
  object_breaking = bit 10 object_colliding = bit 9
  object_exploding = bit 8 ambient = bit 7
```

The named combinations the engine actually uses are the obvious pairings —
`weapon | shooting`, `monster | step`, `anomaly | idle`, `world | ambient`, `item | using`
and so on.

**Invariants**

- **Bit positions are frozen.** These values are written into sound sidecar files shipped
  with the game and are named numerically by scripts, so neither the layout nor the
  assignment may move.
- A well-formed attribute has exactly one source bit and at least one event bit. Nothing
  enforces it; a creature's reaction table simply masks for what it cares about.
- The seven weapon-calibre aliases (`pistol`, `gun`, `submachinegun`, `machinegun`,
  `sniper rifle`, `grenade launcher`, `rocket launcher`) are **all equal to the bare weapon
  bit**. They are vestigial: the AI was once meant to distinguish a pistol shot from a rifle
  shot and never did. A rebuild keeps the names only if it wants the shipped configuration
  spellings to parse.

## the loudness factors

```text
CONSTANT crouch_factor      = 0.3      # a crouching creature's footsteps carry 30% as far
CONSTANT accelerated_factor = 0.5      # ... a running one's, 50%  (see Notes)
```

**Notes** — these two scale a movement sound's AI-audible radius, not its volume. Crouching
at 0.3 is the stealth mechanic in one number. The 0.5 for accelerated movement reads
backwards — running should be *louder* — and the resolution is that the factor is applied to
the sound's own configured range for the accelerated animation set, which is authored louder
to begin with; the factor pulls it back. That is an inference from how it is used and not
stated anywhere, so it belongs on the honest list.

## the editor token table

**Contract** — a parallel table of human-readable names for the useful combinations
("Weapon shooting", "NPC step", "Anomaly idle"), defined in the game module rather than
here. It exists so the level editor can offer a dropdown when an author attaches a sound to
a placed object, and it starts with an "undefined" entry that is *not* one of the values
above. Presentation only.
