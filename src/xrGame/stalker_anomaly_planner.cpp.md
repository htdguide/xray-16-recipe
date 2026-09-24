# src/xrGame/stalker_anomaly_planner.cpp

> The anomaly branch: two propositions, two actions, and the rule that a sub-planner must publish its conclusion to the planner above it.

**Needs** — [`stalker_anomaly_planner.h`](stalker_anomaly_planner.h.md) · [`stalker_anomaly_actions.h`](stalker_anomaly_actions.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — [`stalker_anomaly_planner.h`](stalker_anomaly_planner.h.md)
**Tier floor** — T2: a two-operator search rebuilt on setup.

## Purpose

The smallest complete sub-planner in the stalker, and therefore the clearest example of the
shape every other one follows. It is worth reading before the combat or danger planners,
because the mechanism it shows in eight lines is the same mechanism those hide inside forty
operators.

## State

Two property storages, and the relationship between them is the whole file:

```text
inner  : PropertyStorage    # this planner's own world state, owned
outer  : ref PropertyStorage# the parent planner's world state, borrowed
```

**Invariants** — the proposition `Anomaly` exists in **both**. The inner copy is what this
planner's own actions read and write; the outer copy is what the parent planner branches
on. They are synchronized in one direction only — inner to outer — and only at the end of
each cycle. A rebuild that shares one storage between parent and child instead will find
the parent re-branching in the middle of the child's plan.

## `setup`

**Contract** — bind to the creature and the parent's storage, clear the anomaly proposition
in both, then discard and rebuild the evaluator and operator tables. Idempotent: it is
called again whenever the creature is re-initialized, including after a save is loaded.

```text
FUNCTION setup(creature, parent_storage)
  base.setup(creature, parent_storage)
  inner.Anomaly := false
  outer.Anomaly := false      # the parent must not start believing in a stale anomaly
  clear evaluators and operators
  add_evaluators()
  add_actions()
```

**Invariants** — clearing before adding. Setup rebuilds from scratch rather than patching,
because the alternative is a table that accumulates duplicates across re-initializations.

## `add_evaluators`

**Contract** — install the two questions this branch may ask.

```text
InsideAnomaly : am I physically standing in a hazardous zone
Anomaly       : is there a zone nearby I have not yet probed
```

**Notes** — these are the *only* two facts the branch reasons about. Which zone, how far,
how dangerous — none of it is in the world state; the actions read it off the creature when
they run. Keeping the planner's vocabulary this small is what makes planning per creature
per cycle affordable, and it is the design rule the whole brain follows.

## `add_actions`

**Contract** — install the two operators, with the preconditions and effects that order
them.

```text
GetOutOfAnomaly  requires  InsideAnomaly = true
                 effects   InsideAnomaly = false

DetectAnomaly    requires  InsideAnomaly = false,  Anomaly = true
                 effects   Anomaly = false
```

**Invariants** — the precondition `InsideAnomaly = false` on the probe is what makes the
two mutually exclusive and gives the priority for free: a creature standing in an anomaly
leaves before it investigates anything. No priority number appears anywhere.

**Notes** — the branch's goal is supplied by the parent, which asks for `Anomaly = false`.
Both operators reach it: leaving the zone satisfies the parent because the parent's
evaluator stops reporting an anomaly, and probing satisfies it by declaration. That is why
`GetOutOfAnomaly` claims only the inner proposition and still ends the branch.

## `update`

**Contract** — run one planning-and-execution cycle, then copy the inner anomaly
proposition into the parent's storage.

```text
FUNCTION update()
  base.update()
  outer.Anomaly := inner.Anomaly
```

**Invariants** — the publish happens **after** the cycle, never before and never during.
The parent planner reads its storage on its own cycle, so publishing mid-plan would let the
parent abandon this branch while one of its actions was still configured on the creature.

**Notes** — this two-line function is the contract every planner-as-action owes its parent,
and the reason it has to be written out by hand in each sub-planner is that each publishes a
different proposition. A rebuild can state it once — "a sub-planner declares which of its
propositions is its result, and the framework publishes it" — and delete the override from
every sub-planner in the tree.
