# src/xrGame/stalker_base_action.cpp

> The two rules every stalker action obeys on entry and exit, put in one place so no individual action has to remember them.

**Needs** — [`stalker_base_action.h`](stalker_base_action.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`stalker_planner.h`](stalker_planner.h.md) · [`script_game_object.h`](script_game_object.h.md)
**Used by** — [`stalker_base_action.h`](stalker_base_action.h.md)
**Tier floor** — T2: two method calls per action transition.

## Purpose

An action in a stalker's plan is entered, executed some number of times, and left; the
planner may switch actions at any cycle, without warning and without the action having
reached any conclusion it likes. This file states what must be true across that boundary,
for every action, so that an abandoned action cannot leave the creature wearing something
it set.

There is no state here. The file is three tiny overrides and exists purely so the rules
are stated once.

## State

`Stateless.`

## `initialize`

**Contract** — runs when the planner selects this action. After the base action machinery
has run, it does two things: drops any queued script animations on the creature, and clears
the brain's cover-affecting flag.

```text
FUNCTION initialize()
  base.initialize()
  object.animation.clear_script_animations()
  object.brain.affect_cover(false)
```

**Invariants** — after entry, the creature has no script-queued animation pending and the
cover flag is false. An action that wants either must set it *after* calling up, which is
why the inherited call comes first.

**Notes** — the script-animation queue is a modding surface: Lua may push a sequence of
animations onto a creature. If a plan switch happened while such a queue were still
pending, the new action's animation would fight the leftover queue and the creature would
visibly stutter, so the queue is cleared at both ends of every action rather than trusted
to drain. The cover flag is cleared rather than preserved on the same principle: it is the
*current* activity that decides whether squad cover bookkeeping should account for this
creature, so a stale true from a previous action would misreport the squad's cover state.

## `execute`

**Contract** — pure delegation to the base action machinery. Present only so that every
action's `execute` has a uniform place to call up into; this file adds nothing.

## `finalize`

**Contract** — runs when the planner leaves this action. Clears the script-animation queue
again.

```text
FUNCTION finalize()
  base.finalize()
  object.animation.clear_script_animations()
```

**Notes** — clearing on both entry and exit is not redundancy for its own sake. Exit
clearing covers the action that queued animations and was interrupted; entry clearing
covers the case where the creature was given a queue by something that is not an action at
all (a script call between plan cycles). The cover flag is deliberately *not* cleared on
exit — the next action's entry does that, and an action that ends by handing over to a
successor should not blank the flag in the gap.
