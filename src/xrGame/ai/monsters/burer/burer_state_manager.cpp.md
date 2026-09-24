# src/xrGame/ai/monsters/burer/burer_state_manager.cpp

> The burer's mood chart, with one state no other creature has: a scanning trance it drops into after sensing something at range.

**Needs** — [`burer.h`](burer.h.md) · [`burer_state_manager.h`](burer_state_manager.h.md) · [`burer_state_attack.h`](burer_state_attack.h.md) · [`monster_state_manager.h`](../monster_state_manager.h.md) · [`monster_state_rest.h`](../states/monster_state_rest.h.md) · [`monster_state_panic.h`](../states/monster_state_panic.h.md) · [`monster_state_eat.h`](../states/monster_state_eat.h.md) · [`monster_state_hear_int_sound.h`](../states/monster_state_hear_int_sound.h.md) · [`monster_state_hear_danger_sound.h`](../states/monster_state_hear_danger_sound.h.md) · [`monster_state_hitted.h`](../states/monster_state_hitted.h.md) · [`state_custom_action.h`](../states/state_custom_action.h.md)
**Used by** — [`burer_state_manager.h`](burer_state_manager.h.md)
**Tier floor** — T3: a priority ladder over shared states

## Purpose

Same construction as every creature's — see [`boar_state_manager.cpp`](../boar/boar_state_manager.cpp.md). The burer differs in three ways: its attack slot holds its own tree, its hit reaction is time-boxed, and it has a scanning state built from the generic "perform an action" state rather than from anything burer-specific.

## State

```text
TOP-LEVEL STATES REGISTERED
  rest, panic, eat,
  hear_interesting_sound, hear_dangerous_sound, hit_reaction,
  attack   -> the burer's own attack tree
  scanning -> the generic "perform one action" state, parameterised below
```

```text
scan_state_duration = 4000 ms   # fixed in code
```

## `execute`

```text
FUNCTION execute()
  IF has_enemy()
    state = (danger_from_enemy == strong) ? panic : attack
  ELSE IF was_recently_hit() AND last_hit_time + 10000 ms > now()
    state = hit_reaction
  ELSE IF heard_interesting_sound
    state = hear_interesting_sound
  ELSE IF heard_dangerous_sound
    state = hear_dangerous_sound
  ELSE IF last_scan_time + scan_state_duration > now()
    state = scanning
  ELSE IF hungry_and_a_corpse_is_available()
    state = eat
  ELSE
    state = rest

  select(state)
  current_state.execute()
  previous_substate = current_substate
```

**Notes** — Two details are the burer's own. Its hit reaction carries a second, explicit ten-second window on top of the shared "was I hit" flag: a burer that was shot a long time ago and never found the shooter stops reacting and goes back to what it was doing, where other creatures keep flinching for as long as the flag stands.

The scanning branch is driven from a timestamp the *sensing* ability writes, not from any decision here: when the burer's at-range sense registers something, it stamps the time, and for the next four seconds this ladder puts the creature into a look-around trance. That is what makes a burer telegraph that it has noticed the player before it has seen them.

## `setup_substates`

**Contract** — Fills the generic action state's parameter record when the scanning state is selected: perform the look-around action with idle vocalisations spaced by the creature's configured idle-sound delay. Runs at selection time, not per tick.
