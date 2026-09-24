# src/xrGame/ai/monsters/rats/ai_rat_impl.h

> The nest's budget: how many rats in a group may be awake at once, how many may stand still, and the alarm that spreads a death to all of them.

**Needs** — [`ai_rat.h`](ai_rat.h.md) · [`../../../group_hierarchy_holder.h`](../../../group_hierarchy_holder.h.md) · [`../../../../xrServerEntities/ai_sounds.h`](../../../../xrServerEntities/ai_sounds.h.md)
**Used by** — [`ai_rat.cpp`](ai_rat.cpp.md) · [`ai_rat_behaviour.cpp`](ai_rat_behaviour.cpp.md) · [`ai_rat_fire.cpp`](ai_rat_fire.cpp.md) · [`rat_state_activation.cpp`](rat_state_activation.cpp.md) · [`rat_state_initialize.cpp`](rat_state_initialize.cpp.md) · [`rat_state_switch.cpp`](rat_state_switch.cpp.md)
**Tier floor** — T3: two counters and a proportion test

## Purpose

This is how a nest of forty rats costs what a dozen would. Rats are organised into a
team/squad/group hierarchy whose group record carries three shared counters — how many are
alive, how many are *active*, how many are *standing*. A rat may only become active if the
active count is still below an authored fraction of the living count, and a rat that is not
active is scheduled far less often. The whole crowd effect is four short routines.

The activity budget is also where the two update rates are switched, which makes it the
rat's only concession to the scheduler's degradation rules; every
other creature lets the scheduler decide alone.

## `add_active_member`

**Contract** — asks to join the active budget. Succeeds unconditionally when forced, and
otherwise only if the group's active count is still under its authored share of the living
count. On success: marks the rat active, sets its state to wandering, increments the shared
counter, switches to the fast update interval, and drops out of the standing budget. Does
nothing if the rat is already active.

```text
FUNCTION add_active_member(forced)
  IF is_active  RETURN
  IF forced OR (alive_count * active_percent / 100 >= active_count)
    is_active   = true
    state       = free_active
    active_count = active_count + 1
    schedule_interval = active_interval        # the rat is now expensive
    leave_standing_budget()
```

**Invariants** — the proportion is integer arithmetic on percentages, so a group of three with
a fifty-percent share admits one active rat, not one and a half. **Forced admission ignores the
budget entirely**, which is what the alarm paths rely on: a rat that hears a gunshot, loses its
morale, sees an enemy or notices the nest has moved is forced active regardless of how many are
already running. The budget throttles idle wandering, not reactions.

Joining also *assigns a state directly* rather than pushing one, which is the one place outside
the brain where the rat's state is written. It is why a forced rat is always found wandering
rather than resuming what it was doing.

## `vfRemoveActiveMember`

**Contract** — leaves the active budget: decrements the shared counter, marks the rat inactive,
sets its state to settled, and switches to the slow update interval. Asserts the counter was
positive. Does nothing if already inactive.

**Invariants** — the counter must never go negative, and the assertion is the only thing
enforcing that the join and leave paths are balanced. Death leaves both budgets explicitly,
because a dead rat is not otherwise going to.

## `vfAddStandingMember` / `vfRemoveStandingMember`

**Contract** — the second, independent budget: how many rats may be *standing still* rather
than merely inactive. Same proportion test against the living count, same balanced
increment/decrement, same assertion.

**Notes** — the two budgets look redundant and are not. The active budget caps how many rats
are moving; the standing budget caps how many of the settled ones freeze in place rather than
milling. It is what stops a nest from looking like a photograph — and it is consulted by the
animation selector in [`ai_rat_animations.cpp`](ai_rat_animations.cpp.md), which gives standing
rats a different idle clip.

## `bfCheckIfSoundFrightful`

**Contract** — answers whether the last heard sound was a gunshot or a bullet impact. Pure test
on the recorded sound's kind bits. Used to decide whether a rat that has lost track of its
enemy should keep retreating or turn and fight.

## `update_morale_broadcast`

**Contract** — adds a value to the morale of *every living rat in the group*. Takes a radius and
ignores it.

**Invariants** — the ignored radius is the decision worth recording: a rat's death demoralises
the whole nest **regardless of distance**, so killing rats at the edge of a nest frightens the
ones at the centre. The authored death-broadcast radius is still read from configuration and
passed in, so a rebuild can honour it — but doing so changes the mechanic, because a nest that
only panics locally never breaks.
