# src/xrGame/ai/monsters/rats/ai_rat_feel.cpp

> What a rat is allowed to see, and what hearing something does to its nerve.

**Needs** — [`ai_rat.h`](ai_rat.h.md) · [`../../../memory_manager.h`](../../../memory_manager.h.md) · [`../../../enemy_manager.h`](../../../enemy_manager.h.md) · [`../../../../xrServerEntities/ai_sounds.h`](../../../../xrServerEntities/ai_sounds.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a relevance predicate and an event handler

## Purpose

The rat's half of the senses system. Vision is a one-line filter; hearing is where the rat's
morale actually comes from, which makes this small file the source of most of its behaviour.

## `feel_vision_isRelevant`

**Contract** — asked before the vision system spends a frustum test and a ray on an object.
Admits only living things, and among those, only ones not on the rat's own team — with the
sharp exception that a *dead* teammate is admitted.

```text
FUNCTION feel_vision_isRelevant(object) -> bool
  IF object IS NOT a living-thing type          RETURN false
  IF object.team == my team AND object IS alive RETURN false
  RETURN true
```

**Invariants** — admitting dead teammates is not an oversight; it is how a rat finds a corpse
to eat. The item-valuation in [`ai_rat_fire.cpp`](ai_rat_fire.cpp.md) then decides whether that
particular corpse is acceptable food, and the cannibalism and own-team flags are enforced
*there*, not here. A rebuild that filters teammates out entirely gives rats nothing to scavenge.

## `feel_sound_new`

**Contract** — a sound reached the rat. Takes the emitter, the sound's kind bits, its position
and its loudness. Records the sound if it is loud enough and more significant than what is
already recorded, and adjusts morale by kind. Always forwards to the base afterwards. Ignored
entirely when the rat is dead.

```text
FUNCTION feel_sound_new(who, kind, position, power)
  IF dead  RETURN

  IF kind INCLUDES weapon_shooting
    power = 1.0                    # gunfire is always maximally loud, whatever the distance

  IF power >= sound_threshold
    IF who IS NOT me AND (the recorded sound is stale OR it was quieter than this one)
      record { kind, now, power, position, source: who }

      IF kind INCLUDES monster_dying
        morale = morale + death_quantum
      ELSE IF kind INCLUDES weapon_shooting AND I have no enemy
        morale = morale + fear_quantum
      ELSE IF kind INCLUDES monster_attacking
        morale = morale + success_attack_quantum

  base.feel_sound_new(who, kind, position, power)
```

**Invariants**

- **Gunfire is exempt from distance attenuation.** Its loudness is overwritten with the maximum
  before the threshold test, so a rat hears every shot on the level and is frightened by all of
  them. This single line is why a firefight anywhere sends every nest running, and it is
  deliberate: rats are meant to be an ambient reaction to violence, not a local one.
- **One recorded sound, replaced by loudness.** A rat remembers exactly one sound at a time; a
  newer sound displaces the recorded one only if the recorded one has gone stale (older than the
  last update) or the new one is louder. So a continuous loud source keeps its hold and a
  quieter one cannot interrupt it.
- **The fear penalty from gunfire only applies when the rat has no enemy.** A rat already
  fighting is not further frightened by the shots — otherwise a rat in combat would be
  demoralised by its own target's weapon and always flee.
- **Hearing another creature attack *raises* morale.** The quantum is named for a successful
  attack, and hearing one is taken as the pack doing well. Nothing distinguishes whose attack
  it was, so a rat is emboldened by the player's melee as readily as by its own kind's.

**Notes** — the sign convention is inverted from the obvious one and is easy to get backwards:
the quanta are *added*, and the morale bounds are authored, so whether a quantum frightens or
emboldens depends on its authored sign. The configuration is the only place that decides which
way fear points.

## `feel_touch_on_contact`

**Contract** — pure forward to the base. Declared so the rat's dual inheritance resolves to one
implementation rather than being ambiguous; no rat-specific behaviour.
