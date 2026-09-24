# src/xrGame/stalker_danger_unknown_planner.cpp

> Three steps in a fixed order — cover, sweep, publish — expressed as preconditions rather than as a sequence.

**Needs** — [`stalker_danger_unknown_planner.h`](stalker_danger_unknown_planner.h.md) · [`stalker_danger_unknown_actions.h`](stalker_danger_unknown_actions.h.md) · [`stalker_danger_property_evaluators.h`](stalker_danger_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_danger_unknown_planner.h`](stalker_danger_unknown_planner.h.md)
**Tier floor** — T2: a three-operator search per cycle.

## Purpose

The clearest example in the codebase of a **sequence written as a plan**. The three actions
must happen in one order, and nothing anywhere says so: the order falls out of each action
requiring the previous one's effect. That is worth understanding before reading the combat
planner, which does the same thing with forty operators and is unreadable if you expect a
state machine.

## State

`Stateless.` The three propositions it reasons about live in its own storage; two of them
are *member* propositions, which means they are read from and written to the storage rather
than computed — see below.

## `setup`

**Contract** — bind and rebuild both tables from scratch. Idempotent.

## `initialize`

**Contract** — on entering the branch: release the creature's cover claim with the squad
coordinator, and clear both progress propositions.

**Invariants** — the claim must be released here rather than carried in from whatever the
creature was doing before. The cover appropriate to a fight is not the cover appropriate to
an unlocatable noise, and the cover evaluator's stability logic would otherwise anchor on
a stale choice.

## `add_evaluators`

```text
Danger        : is a danger still selected                    (computed)
CoverActual   : is my chosen cover still the best one         (computed; searches the
                                                               cover database — see
                                                               stalker_danger_property_evaluators.cpp)
CoverReached  : have I arrived                                (stored)
LookedAround  : have I finished sweeping                      (stored)
```

**Invariants** — the last two are *member* evaluators: they do not compute anything, they
read back what an action wrote. That is how an action reports progress to the planner that
is running it, and it is the only channel by which it can, because an action cannot return
a value. A rebuild needs the same split between propositions that are re-derived from the
world every cycle and propositions that are latched by an action.

**Notes** — `CoverActual` is the expensive one, and it is asked every cycle. That cost is
the price of cover selection being *re-validated* continuously rather than decided once:
the world changes, allies claim points, the danger moves. The evaluator's internal
stability rules (an anchor position, a one-unit tolerance) are what keep the continuous
re-validation from producing a continuously changing answer.

## `add_actions`

```text
TakeCover    requires  (nothing)
             effects   CoverActual = true,  CoverReached = true

LookAround   requires  CoverActual = true,  CoverReached = true,  LookedAround = false
             effects   LookedAround = true

Search       requires  CoverActual = true,  CoverReached = true,  LookedAround = true
             effects   Danger = false
```

**Invariants** — the chain is total: the parent's goal is `Danger = false`, only `Search`
produces it, `Search` needs `LookedAround`, only `LookAround` produces that, and it needs
the two cover propositions that only `TakeCover` produces. So the plan is always the same
three actions, and the planner's job is to notice when it has to *restart* the chain rather
than to choose among alternatives.

`TakeCover` has no preconditions at all, which makes it the universal recovery step: if the
cover point is invalidated mid-sweep — an ally takes it, the creature is pushed off it —
`CoverActual` goes false, the later two actions become unavailable, and the plan
re-enters at the top without anything having to detect the failure.

**Notes** — `TakeCover` claims `CoverActual = true` as an effect even though it does not
control the cover evaluator's answer. This is the planner idiom for "running this action is
what makes the world satisfy that proposition": the action moves the creature to the point
the evaluator would choose, and the evaluator then agrees. Effects in this planner are
*claims about the world after the action*, not assignments, and only the latched
propositions are actually written by the action.

## `update` and `finalize`

**Contract** — pure delegation. Unlike the anomaly planner, this one publishes nothing
upward: its terminating action ends the branch by changing the world (publishing a danger
location) rather than by writing the parent's proposition, so there is nothing to
synchronize.
