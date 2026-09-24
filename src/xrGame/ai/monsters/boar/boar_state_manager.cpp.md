# src/xrGame/ai/monsters/boar/boar_state_manager.cpp

> The boar's mood chart: nine top-level states, re-decided from scratch every tick, with the ordering of the tests as the whole of the behaviour.

**Needs** — [`boar.h`](boar.h.md) · [`boar_state_manager.h`](boar_state_manager.h.md) · [`monster_state_manager.h`](../monster_state_manager.h.md) · [`monster_state_rest.h`](../states/monster_state_rest.h.md) · [`monster_state_attack.h`](../states/monster_state_attack.h.md) · [`monster_state_panic.h`](../states/monster_state_panic.h.md) · [`monster_state_eat.h`](../states/monster_state_eat.h.md) · [`monster_state_hear_int_sound.h`](../states/monster_state_hear_int_sound.h.md) · [`monster_state_hear_danger_sound.h`](../states/monster_state_hear_danger_sound.h.md) · [`monster_state_hitted.h`](../states/monster_state_hitted.h.md) · [`monster_state_controlled.h`](../states/monster_state_controlled.h.md) · [`monster_state_help_sound.h`](../states/monster_state_help_sound.h.md)
**Used by** — [`boar_state_manager.h`](boar_state_manager.h.md)
**Tier floor** — T3: a priority ladder over shared states

## Purpose

The boar is a "data plus nothing" creature: it reuses the shared creature states unmodified and differs from its neighbours only in which ones it registers and in what order it prefers them. This file is therefore entirely a table and a ladder.

## State

Stateless beyond the shared machinery's current/previous substate markers.

```text
TOP-LEVEL STATES REGISTERED
  rest, panic, attack, eat,
  hear_interesting_sound, hear_dangerous_sound,
  hit_reaction, controlled, hear_help_sound
```

## `BoarStateManager` construction

**Contract** — Registers the nine shared states against this creature type. Each is allocated once and owned for the creature's lifetime.

## `execute`

**Contract** — One decision per tick, then one substate tick. Selecting a state that is already current is a no-op; selecting a different one tears the old one down through its abort path and initialises the new one. Never blocks, never allocates.

```text
FUNCTION execute()
  IF being_controlled_by_another_creature()
    state = controlled
  ELSE IF has_enemy()
    state = (danger_from_enemy == strong) ? panic : attack
  ELSE IF was_recently_hit()
    state = hit_reaction
  ELSE IF a squadmate's call for help is pending
    state = hear_help_sound
  ELSE IF heard_interesting_sound
    state = hear_interesting_sound
  ELSE IF heard_dangerous_sound
    state = hear_dangerous_sound
  ELSE IF hungry_and_a_corpse_is_available()
    state = eat
  ELSE
    state = rest

  select(state)
  current_state.execute()
  previous_substate = current_substate
```

**Invariants** — Exactly one top-level state is current after this returns; being controlled outranks everything, because a controlled creature's own judgement must not fight the controller's.

**Notes** — The ladder *is* the boar's personality and the only thing that distinguishes it from, say, the cat: a boar puts the call-for-help above both sound reactions, so a herd converges; the interesting sound outranks the dangerous one, so a boar investigates before it flinches. Rewriting the ladder in a different order compiles fine and produces a different animal. Note also that the enemy branch reads a *danger classification*, not a distance or a health fraction — the creature panics from a threat graded strong and fights one graded weak, and that grading lives in the shared enemy-tracking machinery, not here.
