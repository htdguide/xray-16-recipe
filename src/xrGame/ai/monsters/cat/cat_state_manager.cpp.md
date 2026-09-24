# src/xrGame/ai/monsters/cat/cat_state_manager.cpp

> The cat's mood chart: the same shared states as the boar, a different priority ladder, and no controlled state.

**Needs** — [`cat.h`](cat.h.md) · [`cat_state_manager.h`](cat_state_manager.h.md) · [`monster_state_manager.h`](../monster_state_manager.h.md) · [`monster_state_rest.h`](../states/monster_state_rest.h.md) · [`monster_state_attack.h`](../states/monster_state_attack.h.md) · [`monster_state_panic.h`](../states/monster_state_panic.h.md) · [`monster_state_eat.h`](../states/monster_state_eat.h.md) · [`monster_state_hear_int_sound.h`](../states/monster_state_hear_int_sound.h.md) · [`monster_state_hear_danger_sound.h`](../states/monster_state_hear_danger_sound.h.md) · [`monster_state_hitted.h`](../states/monster_state_hitted.h.md) · [`monster_state_help_sound.h`](../states/monster_state_help_sound.h.md) · [`state_test_look_actor.h`](../states/state_test_look_actor.h.md)
**Used by** — [`cat_state_manager.h`](cat_state_manager.h.md)
**Tier floor** — T3: a priority ladder over shared states

## Purpose

The same construction as [`boar_state_manager.cpp`](../boar/boar_state_manager.cpp.md): register the shared top-level states, then pick one per tick. Only the registrations and the order of the tests differ, and those differences are the cat.

## State

```text
TOP-LEVEL STATES REGISTERED
  rest, panic, attack, eat,
  hear_interesting_sound, hear_dangerous_sound,
  hit_reaction, hear_help_sound,
  threaten   -> bound to the shared "stand and look at the player" state
```

```text
RECORD CatStateManager
  last_rotation_jump_time : int   # written once at construction, never read again
```

## `CatStateManager` construction

**Contract** — Registers the nine states above and zeroes `last_rotation_jump_time`.

**Notes** — Two registrations are worth flagging. There is no *controlled* state: a cat cannot be taken over by a controller creature, and its tick has no branch for it. And the threaten slot is filled with the generic "look at the player" state rather than any cat-specific behaviour — the slot exists so that data and debugging tools can name it, but nothing in the tick ever selects it.

## `execute`

**Contract** — One decision per tick, then one substate tick. Same contract as every creature's.

```text
FUNCTION execute()
  IF has_enemy()
    state = (danger_from_enemy == strong) ? panic : attack
  ELSE IF was_recently_hit()
    state = hit_reaction
  ELSE IF a squadmate's call for help is pending
    state = hear_help_sound
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

**Notes** — Against the boar's ladder the cat swaps the two sound reactions: a dangerous sound outranks an interesting one, so a cat flinches first and investigates second. That single swap is most of what separates a skittish animal from a curious one, and it is the kind of decision that a rebuild will get silently wrong if it treats these ladders as boilerplate.

`last_rotation_jump_time` and the three-second spacing constant beside it belong to the retired jump-turn path described in [`cat.cpp`](cat.cpp.md); neither is read. They are recorded here only so a rebuilder does not go looking for the reader.
