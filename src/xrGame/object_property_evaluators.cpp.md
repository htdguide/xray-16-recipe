# src/xrGame/object_property_evaluators.cpp

> The planner's eyes: eleven small observers that turn the live state of a weapon or grenade into the booleans the object-handling search runs on.

**Needs** — [`object_property_evaluators.h`](object_property_evaluators.h.md) · [`Weapon.h`](Weapon.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`Missile.h`](Missile.h.md) · [`FoodItem.h`](FoodItem.h.md) · [`Inventory.h`](Inventory.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-search state queries against live objects

## Purpose

An evaluator is the planner's only channel to reality: it answers one yes-or-no question about
the world, and the search uses the set of answers as its start state. This file holds the
answers for the object-handling planner.

Every one of them is three lines, and the value is entirely in *which* condition each chose.
Several of them fold several underlying states into one boolean in ways that are not obvious,
and those folds are the specification of what the planner believes.

## State

`Stateless` — each evaluator holds the item and the creature, and reads.

## `CObjectPropertyEvaluatorWeaponHidden`

**Contract** — answers true when the weapon is **not** the active item, **or** when it is but
is still playing its draw animation.

```text
RETURN (this weapon is not the active item) OR (its state is "showing")
```

**Invariants** — a weapon mid-draw counts as hidden. That is the load-bearing fold: it stops
the planner from believing the weapon is available and planning a shot that would be issued
into a draw animation. The draw completes, the property flips, and the plan proceeds — one
update later, which is invisible and correct.

## `CObjectPropertyEvaluatorMissileHidden`

**Contract** — the same question for a thrown object, folding four cases: true when nothing is
in hand, when something else is in hand, when this object is in its hidden state, or when it
is still being drawn.

**Notes** — it tests the hidden state explicitly where the weapon version does not, because a
thrown object's state machine has a hidden state it can sit in while still being the active
item. The two are the same idea expressed against two slightly different state machines; a
rebuild unifying the state machines gets one evaluator.

## `CObjectPropertyEvaluatorNoItems`

**Contract** — answers whether the creature's hands are effectively empty: true when there is
no active item, when the active item has no in-hand representation or that representation is
hidden, or when it is still being drawn.

```text
RETURN no active item
    OR the active item has no in-hand form
    OR that form is hidden
    OR that form is being drawn
```

**Invariants** — "being drawn counts as empty" is the same fold as the two hidden evaluators,
applied to the creature rather than to an item, and it must agree with them or the planner
can simultaneously believe a weapon is hidden and that the hands are not empty, which makes
the item-selector property unsatisfiable and the plan fail.

An item with no in-hand form — a bandage, an artefact — counts as empty hands. Only things
that actually occupy the hands count.

## The four magazine evaluators

**Contract** — `Ammo` answers whether the creature carries any suitable ammunition at all;
`Empty`, whether the magazine has no rounds; `Full`, whether the magazine is at capacity;
`Ready`, whether the weapon can fire right now.

```text
ammo   := the total suitable ammunition the creature carries is non-zero
empty  := rounds in the magazine == 0
full   := rounds in the magazine == magazine capacity
ready  := NOT misfired AND rounds in the magazine > 0 AND state is not "reloading"
```

**Invariants** — **all four answer a constant false for the second barrel.** Each is
constructed with a barrel index and each returns false whenever that index is not zero. So
the entire second-barrel half of the planner's world model — the `ammo2`, `empty2`, `full2`,
`ready2` properties — is permanently false, and any plan requiring them is unreachable. That
is the reason a creature never independently manages an underbarrel grenade launcher's
ammunition: the operators exist, the evaluators refuse. A rebuild wanting working second
barrels must implement these four, and the planner's operator table is already complete
enough to use them.

The readiness test folds the reload state in. Its earlier form, preserved commented-out in the
source, checked only misfire and round count; the reload condition was added because a weapon
mid-reload has rounds in the magazine for part of the animation, and without the condition a
creature interrupts its own reload to fire, then reloads again.

Misfire is checked only by readiness. A misfired weapon reads as non-empty and possibly full,
but not ready — which is what drives the plan toward the reload that clears the misfire.

## `CObjectPropertyEvaluatorQueue`

**Contract** — answers whether the weapon is *not* stopped after a fired burst. Answers true
unconditionally for a weapon that is not magazine-fed.

**Invariants** — the polarity is inverted from the property's name: the property is
"queue wait", and it is true when the creature is *free to proceed*, not when it is waiting.
The firing operators require it true as a precondition, and the queue-wait operator's effect
sets it — so the plan reads "wait out the burst, then fire". A rebuild inverting it will make
creatures fire continuously.

A non-magazine weapon has no burst concept, so answering true makes the burst machinery
transparent for it.

## `CObjectPropertyEvaluatorState` and `CObjectPropertyEvaluatorMissile`

**Contract** — the generic form: answers whether the item's state equals a named one, with a
flag that inverts the comparison. Two copies of one idea, one per item kind, because the two
state enumerations are unrelated types.

```text
RETURN (item state == named state) == equality_flag
```

**Notes** — the weapon form is installed nowhere; the line that would have installed it is
commented out in favour of the dedicated hidden evaluator. The thrown-object form is installed
once, to observe the throw-end state. The inversion flag is never used with anything but its
default. A rebuild needs only the one installed use and can drop the generality.

## `CObjectPropertyEvaluatorMissileStarted`

**Contract** — answers whether the thrown object is in its throwing state.

**Notes** — it is the generic state evaluator with the state fixed and the inversion dropped,
which is why it looks redundant next to it. It exists as a named class so the planner's
installation reads as a question rather than as a state constant.
