# src/xrGame/object_handler_planner_missile.cpp

> The thrown-object handling model as planner data: six evaluators and six operators, and a throw that is a three-stage gesture.

**Needs** — [`object_handler_planner.h`](object_handler_planner.h.md) · [`object_handler_planner_impl.h`](object_handler_planner_impl.h.md) · [`object_handler_space.h`](object_handler_space.h.md) · [`object_property_evaluators.h`](object_property_evaluators.h.md) · [`object_actions.h`](object_actions.h.md) · [`Missile.h`](Missile.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: declarative planner-set construction

## Purpose

The thrown-object counterpart to
[`object_handler_planner_weapon.cpp`](object_handler_planner_weapon.cpp.md), and a fraction of
its size — a grenade has no magazine, no fire mode and no sling. Read as data, it is the
specification of how the engine models throwing something: pull the pin, wind up, release,
and optionally stop at the wind-up to threaten rather than throw.

## State

`Stateless.`

## `add_evaluators`

**Contract** — installs six evaluators for one thrown object.

**Observed**:

```text
  hidden          # is it out of the creature's hands
  throw_started   # has the wind-up begun
  throw           # has the object's own state machine reached its throw-end state
```

**Constant false**:

```text
  dropped, firing1, idle
```

**Invariants** — the same rule as on the weapon side: a property meaning "I did this" gets a
constant-false evaluator, so the plan must contain the operator that produces it. The reuse of
`firing1` for the *threaten* outcome is why it appears here at all — a thrown object has no
barrel, and this property is being borrowed to mean "the threatening gesture has been made".

The `throw` property's evaluator watches for the object's **throw-end** state, not its throw
state. The plan is considered to have thrown once the release has completed, not once it has
begun.

**Notes** — an evaluator for the throw-idle property is written and commented out, and so is a
constant-false evaluator for slung and for aiming-ready. The throw-idle property has no
evaluator at all, which means it is never observable; only the operator's inertia time (set at
the end of the operator function) gives it any presence in the model. See the note there.

## `add_operators`

**Contract** — installs six operators.

```text
show          needs: hidden, and no other item selected
              gives: not hidden, this item selected

hide          needs: not hidden, this item selected
              gives: hidden, no item selected

drop          needs: not hidden
              gives: dropped

idle          needs: not hidden
              gives: idle, throw_started cleared, firing1 cleared

throw_start   needs: not hidden, not already started
              gives: throw_started                    # inertia 1500 ms

throw         needs: not hidden, throw_started, not yet thrown
              gives: throw

threaten      needs: thrown, not yet "firing1"
              gives: firing1
```

**Invariants** — the throw is a strict three-stage chain, and the chain's order is the whole
model: *start* begins the wind-up, *throw* releases, *threaten* is the optional coda. Each
stage's precondition is the previous stage's effect, so the planner cannot skip or reorder
them, and a plan interrupted between stages leaves the creature holding a live grenade in a
defined state.

The *idle* operator clears both throw-started and firing, which is the reset: returning a
thrown-object plan to idle must abandon a half-made gesture rather than resume it. Its
implementing action class clears the same three properties directly at setup, so the reset
happens both as a planner effect and as a real side effect.

Unlike the weapon set, none of these operators has a "not slung" precondition. A thrown object
cannot be slung, so the property never becomes true for it and the precondition would be dead
weight.

**Notes** — the *threaten* operator produces `firing1` from `throw`, which reads backwards: it
says the threatening gesture is available only *after* the object has been thrown. Given that
threatening means brandishing rather than throwing, the intended ordering was presumably the
reverse. The *show*, *throw* and *threaten* operators are all instances of the plain action
base, meaning they have no behaviour of their own and exist purely as state transitions that
the object's own animation drives; so the discrepancy affects only which order the planner will
sequence them in.

The wind-up operator's inertia time is 1500 milliseconds by default, and is then overwritten at
setup by the throw operator itself, which scales it between 1000 and 2500 milliseconds by the
distance to the target (see [`object_actions.cpp`](object_actions.cpp.md)). The default is what
applies for the one frame before the operator runs.

The last line sets a 2000-millisecond inertia on the **throw-idle** operator — which is never
installed by this function, and whose property has no evaluator. It reaches into the planner's
operator table by identifier for an operator that does not exist. Whatever stage it belonged to
was removed; the line and the two commented-out evaluator lines are all that remain of it. A
rebuild should omit it.
