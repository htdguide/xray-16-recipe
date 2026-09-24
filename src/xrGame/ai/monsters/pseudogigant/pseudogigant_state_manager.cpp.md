# src/xrGame/ai/monsters/pseudogigant/pseudogigant_state_manager.cpp

> The giant's brain, which is the chapter's baseline selector with one branch added: being someone else's puppet outranks everything.

**Needs** — [`pseudogigant_state_manager.h`](pseudogigant_state_manager.h.md) · [`pseudo_gigant.h`](pseudo_gigant.h.md) · [`../states/monster_state_rest.h`](../states/monster_state_rest.h.md) · [`../states/monster_state_attack.h`](../states/monster_state_attack.h.md) · [`../states/monster_state_panic.h`](../states/monster_state_panic.h.md) · [`../states/monster_state_eat.h`](../states/monster_state_eat.h.md) · [`../states/monster_state_hitted.h`](../states/monster_state_hitted.h.md) · [`../states/monster_state_hear_int_sound.h`](../states/monster_state_hear_int_sound.h.md) · [`../states/monster_state_hear_danger_sound.h`](../states/monster_state_hear_danger_sound.h.md) · [`../states/monster_state_help_sound.h`](../states/monster_state_help_sound.h.md) · [`../states/monster_state_controlled.h`](../states/monster_state_controlled.h.md)
**Used by** — [`pseudogigant_state_manager.h`](pseudogigant_state_manager.h.md)
**Tier floor** — T3: a seven-branch priority chain

## Purpose

Nine states, every one of them the shared implementation instantiated for this creature, and a
flat priority chain. Nothing about the giant's weight, its stomp or its shake appears here:
those are abilities the shared base offers and the creature gates, not states in the tree. This
file is worth reading precisely because it is so nearly empty — it shows how much of a creature
the chapter expects to live in data and in ability hooks rather than in a brain.

## `execute` — the selector

**Contract** — chooses a state, runs it, records it as previous. Called every tick through the
root's update guards.

```text
FUNCTION execute()
  IF self IS under someone else's control
    state = controlled
  ELSE
    enemy = enemy_manager.enemy
    IF enemy EXISTS
      state = CASE enemy_manager.danger_type OF
                strong : panic         # the odds are bad
                weak   : attack        # the odds are good
    ELSE IF hit_memory.has_hits()
      state = react_to_hit
    ELSE IF a call for help is pending
      state = answer_help_call
    ELSE IF heard_interesting_sound
      state = investigate_sound
    ELSE IF heard_dangerous_sound
      state = react_to_danger_sound
    ELSE IF can_eat()
      state = eat
    ELSE
      state = rest

  select_state(state)
  current_state.run()
  previous_substate = current_substate
```

**Invariants**

- **Being controlled is tested first and shortcuts the whole chain**, so a puppeted giant
  ignores its enemy, its wounds and every sound. That is the point: a controller that takes a
  giant gets a giant that does what the controller says and nothing else.
- **The chain is re-evaluated from the top every tick with no continue-or-start test at any
  level.** Persistence, where the giant has any, lives inside the chosen state's own selector.
  A giant that acquires an enemy abandons its meal in the same tick.
- **Interesting sound outranks dangerous sound** here, as it does for the dog, so a giant walks
  toward gunfire rather than away from it. The snork inverts this pair; the giant does not.
- **Answering a call for help outranks investigating a sound**, which is what keeps a pack
  coherent: a squadmate's call pulls the giant before a random noise does.

**Notes** — the danger-type branch has no default arm. The enemy manager's "no verdict yet"
value, produced on the tick a target is first acquired and for as long as a target is *forced*
by a controller or a script, leaves the choice unassigned and the tick then fails on an
unregistered identifier. Nothing guards it. In practice the verdict is set on the same tick as
the enemy, which is why this has never surfaced — but the controlled branch above forces an
enemy, so the safety margin is thinner than it looks. A rebuild should default the branch to
the attack state.

The giant's motion-control abilities — the run-attack, the stomp and the rotation jump — are
registered on the creature, not here, and are offered to whatever state is running. That is why
a giant can stomp while its brain believes it is simply attacking.
