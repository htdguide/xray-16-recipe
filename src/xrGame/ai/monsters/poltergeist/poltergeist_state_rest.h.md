# src/xrGame/ai/monsters/poltergeist/poltergeist_state_rest.h

> The poltergeist's idle state: the shared resting behaviour with its eating and sleeping branches removed, leaving a strict three-step priority over where to be.

**Needs** — [`../states/monster_state_rest.h`](../states/monster_state_rest.h.md) · [`../state.h`](../state.h.md)
**Used by** — [`poltergeist_state_manager.cpp`](poltergeist_state_manager.cpp.md)
**Tier floor** — T3: a three-test selector

## Purpose

A creature's resting state is a selector like any other, and most creatures use the shared one.
The poltergeist replaces only its selector, keeping the shared state's registered children.

What the replacement removes is every branch that depends on having a body — the shared
resting state's sleeping and feeding behaviour — and what it leaves is a strict priority over
*where the creature should be*, which is all a drifting presence has to decide.

## The selector

**Contract** — runs each tick while resting. Chooses among four sub-states by a fixed priority,
using the start-or-continue rule at each step, then runs the chosen one.

```text
FUNCTION execute()
  IF should_run(smart_terrain_task)
    select smart_terrain_task                    # an authored place has a job for us
  ELSE IF should_run(move_to_restrictor)
    select move_to_restrictor                    # we are outside our permitted volume
  ELSE IF should_run(move_to_home_point)
    select move_to_home_point                    # we are outside our home
  ELSE
    select walk_graph_point                      # drift to a random nearby graph point

  current_state.run()
  previous_substate = current_substate
```

**Invariants** — the priority is absolute and each test is the start-or-continue rule from
[`../monster_state_manager.h`](../monster_state_manager.h.md). The two interact in one
direction only: persistence keeps a sub-state running against *weaker* claims, and a *stronger*
claim preempts it regardless. A creature part-way to its restrictor keeps going while nothing
but home or wandering competes, and drops it the moment a smart terrain offers a job.

**Notes** — the order is the design: an authored job outranks staying inside the permitted
volume, which outranks being at home, which outranks wandering. A creature is only ever
wandering because none of the three stronger claims applies.

The final branch has no test at all — wandering is the state with no conditions, the floor of
the priority — which is why the whole chain terminates.

The deep nesting in the original is four levels of else, one per test; it is a chain, not a
tree, and a rebuild should write it as one.

This same three-step priority appears, with small variations, in most creatures' resting
states. What is distinctive here is only what is *absent*: no eating, no sleeping, no posture.
