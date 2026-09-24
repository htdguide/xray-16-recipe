# src/xrGame/stalker_search_planner.cpp

> What a stalker does after the enemy disappears: walk to where he was, find a spot that
> watches it, and wait there.

**Needs** — [`stalker_search_planner.h`](stalker_search_planner.h.md) · [`stalker_search_actions.h`](stalker_search_actions.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md)
**Used by** — [`stalker_search_planner.h`](stalker_search_planner.h.md)
**Tier floor** — T2: three operators over three propositions.

## Purpose

A sub-planner of the combat planner, entered when the enemy has been lost. It is the
smallest interesting planner in the stalker tree — three propositions, three operators, one
goal — and it is a good place to read how the tree's levels compose, because everything
about it is inherited from its parent except the sequence it imposes.

The whole planner exists to make a *sequence* out of what would otherwise be three
independent actions. Its three operators form a chain in which each one's effect is the
next one's precondition, so the plan search can only ever produce one order.

## The world model

```text
PureEnemy              : constant true    # the goal proposition, never satisfied by measurement
EnemyLocationReached   : read from the parent planner's world state
AmbushLocationReached  : read from the parent planner's world state
```

Only one of these is measured, and it is measured as a constant. `PureEnemy` — *is there an
enemy right now* — is forced to **true** here, which is the opposite of what the same
property means one level up. That inversion is the design: the search planner is entered
precisely because the enemy is gone, so measuring the property honestly would satisfy the
goal immediately and the planner would do nothing. Forcing it true makes "there is no longer
an enemy to search for" the goal that only the last operator in the chain can deliver.

The other two are read out of the parent's world state rather than measured, which is what
lets the parent reset them (see `initialize`) and what lets progress survive a re-plan
inside this planner.

## The three operators

```text
ReachEnemyLocation   requires  EnemyLocationReached  = false
                     effects   EnemyLocationReached  = true

ReachAmbushLocation  requires  EnemyLocationReached  = true
                               AmbushLocationReached = false
                     effects   AmbushLocationReached = true

HoldAmbushLocation   requires  AmbushLocationReached = true
                     effects   PureEnemy             = false
```

Read as a chain: go to where he was; from there find a spot that covers it; sit in that spot
until the enemy is considered gone. There is no branch and no alternative route to the goal,
which is why this is a planner at all rather than a three-state machine — the planner
machinery is already there, and expressing the order as preconditions means a mid-sequence
disturbance (a hit, a re-plan) resumes at the right step rather than restarting.

`HoldAmbushLocation` carries an **inertia time of fifteen seconds**: once chosen, the
planner will not switch away from it for that long even if another operator becomes
preferable. Without it a creature whose ambush spot is marginal oscillates between holding
and re-seeking, visibly twitching. Fifteen seconds is also roughly how long the hold action
itself needs before it reaches its own give-up condition, so the two agree.

## `setup`

**Contract** — binds to a stalker and to a property storage, clears any previous tables, and
installs the three evaluators and three operators above. Rebuilding from scratch on each
call is deliberate: this planner is re-set-up whenever the creature is re-initialized.

**Invariants** — evaluators before operators; an operator whose precondition names a
property with no evaluator makes every plan fail.

## `initialize`

**Contract** — run once when the parent planner selects this branch. Three effects, and the
order does not matter but the presence of all three does:

```text
FUNCTION initialize()
  release_my_squad_cover()                        # I am moving; my reserved cover is free
  parent_state.EnemyLocationReached  := false
  parent_state.AmbushLocationReached := false
```

Clearing the two progress flags is what makes the search repeatable: the second time a
stalker loses the same enemy it walks the chain again from the start rather than finding
both flags still set from the last search and going straight to holding.

Releasing the cover reservation is the squad-facing half. A stalker that is about to walk
somewhere must stop holding a cover slot, or the squad's cover bookkeeping will keep
steering other members away from a spot nobody is standing in.

## `update` / `finalize`

**Contract** — pure delegation to the planner base. They exist as override points and carry
no decision; a rebuild need not have them at all.
