# src/xrGame/ai/monsters/chimera/chimera_state_manager.cpp

> The chimera's mood chart: six states, and a creature that never flinches from being shot.

**Needs** — [`chimera.h`](chimera.h.md) · [`chimera_state_manager.h`](chimera_state_manager.h.md) · [`chimera_attack_state.h`](chimera_attack_state.h.md) · [`monster_state_manager.h`](../monster_state_manager.h.md) · [`monster_state_rest.h`](../states/monster_state_rest.h.md) · [`monster_state_panic.h`](../states/monster_state_panic.h.md) · [`monster_state_eat.h`](../states/monster_state_eat.h.md) · [`monster_state_hear_int_sound.h`](../states/monster_state_hear_int_sound.h.md) · [`monster_state_hear_danger_sound.h`](../states/monster_state_hear_danger_sound.h.md)
**Used by** — [`chimera_state_manager.h`](chimera_state_manager.h.md)
**Tier floor** — T3: a priority ladder over shared states

## Purpose

Same construction as every creature's — see [`boar_state_manager.cpp`](../boar/boar_state_manager.cpp.md). What distinguishes the chimera is how *little* it registers.

## State

```text
TOP-LEVEL STATES REGISTERED
  rest, panic, eat,
  hear_interesting_sound, hear_dangerous_sound,
  attack -> the chimera's own pounce-driven attack state
```

## `ChimeraStateManager` construction

**Contract** — Registers six states. The attack slot is filled with the creature's own state, [`chimera_attack_state_inline.h`](chimera_attack_state_inline.h.md), rather than the shared one — the chimera does not chase and bite, it circles and pounces, and the shared attack cannot express that.

**Notes** — Three registrations the other creatures have are deliberately absent, and each absence is behaviour. There is no *hit reaction*: shooting a chimera does not interrupt it. There is no *threaten*: the subtree for it exists in this directory and is never instantiated — see [`chimera_state_threaten_inline.h`](chimera_state_threaten_inline.h.md). And there is no *hear help sound*: chimeras do not converge on each other's calls.

## `execute`

**Contract** — One decision per tick, then one substate tick.

```text
FUNCTION execute()
  IF has_enemy()
    state = (danger_from_enemy == strong) ? panic : attack
  ELSE IF heard_dangerous_sound
    state = hear_dangerous_sound
  ELSE IF heard_interesting_sound
    state = hear_interesting_sound
  ELSE IF hungry_and_a_corpse_is_available()
    state = eat
  ELSE
    state = rest

  select(state)
  current_state.execute()
  previous_substate = current_substate
```

**Notes** — Without a hit-reaction branch, a chimera with no enemy that takes fire from an unseen shooter falls through to the sound branches or to rest. Damage alone does not make it hunt; it needs to see or hear the source.
