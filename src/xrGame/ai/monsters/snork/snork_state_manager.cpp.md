# src/xrGame/ai/monsters/snork/snork_state_manager.cpp

> The snork's brain: the baseline selector with caution ahead of curiosity, plus the one line that arms the pre-fight snarl on the tick combat begins.

**Needs** — [`snork_state_manager.h`](snork_state_manager.h.md) · [`snork.h`](snork.h.md) · [`../states/monster_state_rest.h`](../states/monster_state_rest.h.md) · [`../states/monster_state_attack.h`](../states/monster_state_attack.h.md) · [`../states/monster_state_panic.h`](../states/monster_state_panic.h.md) · [`../states/monster_state_eat.h`](../states/monster_state_eat.h.md) · [`../states/monster_state_hitted.h`](../states/monster_state_hitted.h.md) · [`../states/monster_state_hear_int_sound.h`](../states/monster_state_hear_int_sound.h.md) · [`../states/monster_state_hear_danger_sound.h`](../states/monster_state_hear_danger_sound.h.md) · [`../states/monster_state_help_sound.h`](../states/monster_state_help_sound.h.md) · [`../states/state_test_state.h`](../states/state_test_state.h.md)
**Used by** — [`snork_state_manager.h`](snork_state_manager.h.md)
**Tier floor** — T3: a six-branch priority chain and one edge detection

## Purpose

The snork's selector differs from the chapter's baseline in exactly two ways, and both are
worth a sentence.

**Caution outranks curiosity.** Where the dog and the giant investigate an interesting sound
before reacting to a dangerous one, the snork does the reverse. A snork that hears gunfire
takes cover; a dog that hears it comes to look. This single ordering is most of what makes the
two creatures read differently when they are not fighting.

**The snarl is armed on an edge, not on a state.** The selector detects the *transition* into
attack and sets a one-shot flag on the creature; the creature's ability gate then consumes it,
half the time. See [`snork.cpp`](snork.cpp.md).

## `execute` — the selector

**Contract** — chooses a state, runs it, records it as previous. Called every tick through the
root's update guards.

```text
FUNCTION execute()
  enemy = enemy_manager.enemy

  IF enemy EXISTS
    state = CASE enemy_manager.danger_type OF
              strong : panic
              weak   : attack
  ELSE IF hit_memory.has_hits()
    state = react_to_hit
  ELSE IF a call for help is pending
    state = answer_help_call
  ELSE IF heard_dangerous_sound          # caution before curiosity: the snork's own ordering
    state = react_to_danger_sound
  ELSE IF heard_interesting_sound
    state = investigate_sound
  ELSE IF can_eat()
    state = eat
  ELSE
    state = rest

  select_state(state)

  IF current_substate == attack AND current_substate != previous_substate
    self.start_threaten = true           # the edge, not the state: fires once per entry

  current_state.run()
  previous_substate = current_substate
```

**Invariants**

- **The snarl arming is an edge detection and must stay one.** It compares the freshly selected
  state against the previous tick's, so it fires exactly on the transition into combat. Arming
  it while *in* attack would let the snork snarl repeatedly; arming it in the state's own entry
  would be equivalent but would put creature-specific knowledge inside a shared state.
- The chain is re-evaluated from the top every tick with no continue-or-start test, as
  everywhere in the chapter.
- There is **no controlled branch**: a snork cannot be taken over by a controller. It is one of
  the creatures the controller's puppet mechanic does not reach.

**Notes** — the brain registers a **ninth state that is never selected.** A cover-testing
behaviour is registered under the find-enemy identifier, and the only line that would have
chosen it is commented out immediately below the chain. It is a developer harness for
inspecting the cover query — the same query the developer-build drawing in
[`snork.cpp`](snork.cpp.md) visualises — left registered in the shipped build. It costs one
allocation per snork and never runs. A rebuild should drop it.

The danger-type branch has no default arm, with the same consequence as elsewhere in the
chapter: the enemy manager's "no verdict yet" value leaves the choice unassigned and the tick
then fails. A rebuild should default it to the attack state.
