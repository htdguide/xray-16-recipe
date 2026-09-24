# src/xrGame/ai/monsters/pseudodog/psy_dog_aura.cpp

> The psy dog's presence as a feeling rather than an attack: while its illusions and the player are aware of each other and the real creature is close, the player's screen curdles — and it eases off a few seconds after that stops being true.

**Needs** — [`psy_dog_aura.h`](psy_dog_aura.h.md) · [`psy_dog.h`](psy_dog.h.md) · [`Actor.h`](../../../Actor.h.md) · [`ActorEffector.h`](../../../ActorEffector.h.md) · [`actor_memory.h`](../../../actor_memory.h.md) · [`visual_memory_manager.h`](../../../visual_memory_manager.h.md) · [`Level.h`](../../../Level.h.md)
**Used by** — [`psy_dog_aura.h`](psy_dog_aura.h.md)
**Tier floor** — T2: a linear fade factor and two perception scans per scheduled tick

## Purpose

The psy dog fights by sending out illusions of itself while the real animal hides. This file supplies the other half of that design: a screen treatment that tells the player, without a hit or a sound, that the illusions around him are not accidents. It is the only feedback linking the phantoms back to a creature the player may never have seen.

Its whole substance is one predicate — *should the dread be on right now* — and one three-phase fade that keeps the answer from flickering.

## State

```text
RECORD PsyDogAuraEffector
  phase        : ENUM { fading_in, fading_out, holding }
  phase_began  : int          # tick the current phase started
  fade_length  : int          # how long a fade takes, in milliseconds
  factor       : real         # inherited; 0 = no effect, 1 = fully applied

RECORD PsyDogAura
  creature                 : reference to the psy dog
  player                   : reference to the player
  effector                 : optional reference to the live effect
  last_player_saw_phantom  : int    # tick
  last_phantom_saw_player  : int    # tick
  look                     : colour-grading record   # loaded from configuration
```

**Invariants** — the effect reference is non-empty exactly while an effect is live. The controller never destroys the effect: it asks it to fade out and drops its reference, and the camera stack retires it when the fade completes. Every path that stops the aura — the predicate turning false, and the creature dying — does both halves, so a dropped reference never leaves an effect stuck at full strength.

## `PsyDogAuraEffector`

**Contract** — asked for its strength each frame. Ramps linearly from nothing to fully applied over the fade length, then holds. When switched off, it ramps down from wherever it is and reports exhaustion when it reaches nothing, which is how the camera stack knows to retire it.

```text
FUNCTION update() -> bool                   # false means "retire me"
  IF phase is holding
    factor = 1 ; RETURN true

  factor = (now - phase_began) / fade_length
  IF phase is fading_out  factor = 1 - factor

  IF factor > 1
    phase = holding ; factor = 1
  ELSE IF factor < 0
    RETURN false
  RETURN true

FUNCTION switch_off()
  phase = fading_out ; phase_began = now
```

**Notes** — switching off restarts the clock from the current moment rather than from the strength actually reached, so an effect cut short during its fade-in still takes a full fade length to disappear and begins that fade from full strength — the screen briefly gets *worse* before it clears. A rebuild that wants a symmetric fade must carry the reached strength into the fade-out.

## `PsyDogAura.reinit`

**Contract** — clear both timestamps and cache the player. Called when the creature is reinitialised.

**Notes** — the player is cached once, not looked up per tick, and the predicate below tolerates it being absent. The cache is the only reason the per-tick scan is affordable.

## `PsyDogAura.update_schedule` — the predicate

**Contract** — run once per scheduled creature tick. Scan the player's visual memory for any phantom of this creature he can see right now, and scan this creature's phantoms for the most recent moment any of them had the player as an enemy or in its own memory. Then decide whether the aura should be on, and create or retire the effect accordingly. Does nothing if the creature is dead or the player is absent.

```text
FUNCTION update_schedule()
  IF creature is dead OR player is absent  RETURN

  last_phantom_saw_player = 0
  FOR EACH object the player remembers seeing
    IF it is a phantom AND the player can see it right now
      last_player_saw_phantom = now

  FOR EACH phantom this creature owns
    IF its current enemy is the player
      last_phantom_saw_player = now
    ELSE
      last_phantom_saw_player = max(last_phantom_saw_player,
                                    when it last remembered the player)
    IF last_phantom_saw_player == now  BREAK        # cannot get more recent

  close = distance(creature, player) < 30

  want = close AND ( last_player_saw_phantom within the last 2 seconds
                  OR last_phantom_saw_player within the last 10 seconds )

  IF effect is live AND NOT want   effect.switch_off() ; drop the reference
  IF effect is absent AND want     create the effect with a 5-second fade
                                   and hand it to the player's camera stack
```

**Invariants** — the phantom-sightings timestamp is recomputed from scratch every tick; the player-sighting timestamp is *not*, it is only ever moved forward. That asymmetry is what gives the two halves different memories, and it is the reason for the two different windows below.

**Notes** — the two windows are the design. Two seconds is how long the dread outlives the player *seeing* an illusion, which is short because the sight itself is the stronger signal. Ten seconds is how long it outlives an illusion *noticing the player*, which is long because the player has no way to know that happened and needs the effect to persist for it to mean anything. Either alone produces a worse creature: only the first makes the aura a caption on something already visible; only the second makes it arrive from nowhere.

The 30-unit proximity test is on the *real* creature, not the phantoms. So a player surrounded by illusions feels nothing if the animal itself has withdrawn far enough, which is exactly the tell that lets an attentive player work out that the real one is near.

The scan over the creature's phantoms stops as soon as one reports the current tick, because no later answer is possible. The scan over the player's memory has no such exit and walks the whole list; the list is small.

Neither window, nor the 30-unit radius, nor the 5-second fade is read from configuration. The *look* of the effect is; its timing is not.

## `PsyDogAura.on_death`

**Contract** — if an effect is live, switch it off and drop the reference. The dread does not survive the animal.
