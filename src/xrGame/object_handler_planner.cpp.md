# src/xrGame/object_handler_planner.cpp

> Turns "use this object this way" into a target world state, keeps the operator set in step with the inventory, and rolls the creature's burst rhythm.

**Needs** — [`object_handler_planner.h`](object_handler_planner.h.md) · [`object_handler_planner_impl.h`](object_handler_planner_impl.h.md) · [`object_handler_space.h`](object_handler_space.h.md) · [`object_property_evaluators.h`](object_property_evaluators.h.md) · [`object_actions.h`](object_actions.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md) · [`Inventory.h`](Inventory.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`Missile.h`](Missile.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: planner set maintenance and goal translation, once per creature update

## Purpose

The common half of the object-handling planner. It does three things the item-kind siblings do
not: translate the brain's vocabulary of *object actions* into the planner's vocabulary of
*world properties*; add and remove a whole item's worth of operators and evaluators when the
inventory changes; and own the burst schedule, which is the only place in the object-handling
layer where randomness enters.

## State

```text
RECORD ObjectHandlerPlanner                 # extends the generic planner over a stalker
  min_queue_size, max_queue_size   : int    # bounds requested by the caller
  min_queue_interval, max_queue_interval : int (milliseconds)
  queue_size       : int                    # the rolled value currently in force
  queue_interval   : int (milliseconds)
  next_time_change : int (milliseconds)     # when the roll expires
```

## `object_property` — the goal translation

**Contract** — maps one named object action onto the world property whose truth means that
action has been performed. Total over the seventeen actions the brain can ask for; any other
value is a programming error.

```text
  switch1 / switch2          -> switch1 / switch2
  aim1                       -> aiming_ready1          # not "aimed1"
  aim2                       -> aiming2                # not "aiming_ready2"
  aim_ready1 / aim_ready2    -> aiming_ready1 / aiming_ready2
  aim_force_full1 / ...2     -> aim_force_full1 / ...2
  fire1 / fire2              -> firing1 / firing2
  fire_no_reload             -> firing_no_reload1
  idle                       -> idle
  activate                   -> idle
  strapped                   -> idle_strap
  deactivate                 -> the no-item idle sentinel
  drop                       -> dropped
  use                        -> used
```

**Invariants** — the table is the seam between the brain's vocabulary and the planner's, and
two entries in it are asymmetric in a way that matters. Asking for *aim1* yields the
**ready**-to-fire property, so a plain aim order on the first slot implicitly requires the
weapon to be loaded; asking for *aim2* yields the plain **aiming** property, which does not.
The first slot is the primary weapon and a creature aiming it is about to shoot; the second is
a secondary (grenade launcher, underbarrel) that a creature may aim without being loaded.
Whether the asymmetry is intended or an editing slip is not recoverable from the source, but
it is observable behaviour and a rebuild copying the table will reproduce it.

*Activate* and *idle* map to the same property. Activating an item is, in this planner's
terms, exactly "get it into your hands and stand there holding it".

## `set_goal`

**Contract** — installs a new target world state, and re-rolls the burst schedule when the
requested bounds have changed or the previous roll has expired. Takes the object action, the
target object (which may be absent), and four burst bounds.

```text
FUNCTION set_goal(action, object, min_size, max_size, min_interval, max_interval)
  goal := object_property(action)

  IF object exists AND goal is not the no-item sentinel
    IF the object is a weapon that cannot be slung AND goal is "idle_strap"
      goal := "idle"                        # degrade gracefully rather than plan the impossible
    condition := uid(object identifier, goal)
  ELSE
    condition := the no-item idle sentinel

  target state := { condition is true }     # exactly one condition, always

  IF no object, or it is not a magazine-fed weapon THEN RETURN

  IF the bounds differ from those in force, OR the roll has expired
    adopt the new bounds
    queue_size     := max(1, min_size)          IF the bounds are equal
                    | max(1, random in [min_size, max_size])
    queue_interval := min_interval              IF the bounds are equal
                    | random in [min_interval, max_interval]
    next_time_change := now + queue_interval
    tell the weapon its burst size
    set the inertia time of both queue-wait operators to queue_interval,
      or to 300 ms if it is zero
```

**Invariants** — the target state holds **exactly one** condition. The whole object-handling
plan is therefore always a search toward a single fact, which is what makes it cheap enough to
re-plan every update. The item-removal path asserts this invariant explicitly.

The burst size is clamped to at least one. A zero-size burst would be a weapon that never
fires, and the callers do pass zero as "unspecified".

The schedule re-rolls when the bounds change *or* when the interval since the last roll has
elapsed — not on every goal change. So a creature firing repeatedly at the same enemy keeps
one rhythm for one interval and then picks a new one, which is what makes AI gunfire sound
like a person shooting rather than a metronome. A rebuild re-rolling per goal loses that.

**Notes** — the graceful degradation of a slung-idle goal for a weapon that cannot be slung is
the alternative to a plan failure. Pistols cannot be slung; asking a creature to sling one is
a legitimate order from a brain that does not know what it is carrying, and the planner
answers by holding it instead.

The burst interval is also written into the two queue-wait operators' inertia times, which is
how the "wait between bursts" operator knows how long to wait. Zero falls back to 300
milliseconds, the same default the object handler's goal-setting uses.

## `setup`

**Contract** — bind the planner to a creature and bring it to a known state. Zeroes the burst
schedule, clears every operator and evaluator, initializes the shared world storage, installs
the two no-item evaluators and the single no-item idle operator, and sets the goal to idle
with no object.

```text
FUNCTION setup(creature)
  base setup
  zero every burst field
  clear all operators and evaluators
  init_storage()
  evaluator for "no items"      := does the creature have no items?
  evaluator for "no items idle" := constantly false
  operator "no items idle":
    precondition  the no-item item-selector property is true
    effect        the no-item idle property becomes true
  set_goal(idle, no object, 0, 0, 0, 0)
```

**Invariants** — the "no items idle" evaluator is a constant **false**, so that property is
never satisfied by observation and can only be achieved by running its operator. That is how
the planner is made to actually execute the standing-there operator instead of concluding it
is already standing there. The same trick appears throughout the item-kind siblings: any
property whose truth means "I performed an action" gets a constant-false evaluator.

The no-item operator's precondition and effect use the reserved all-ones pseudo-entity, so it
is the one operator in the set not bound to a real item, and the one that is always available.

## `init_storage`

**Contract** — clears five shared, item-independent properties: both aim-completed flags, the
used-enough flag, and both sling flags.

**Invariants** — these five are *not* per-item. They describe the creature's hands, which are
singular, and they are the properties the operators write directly rather than deriving from
an evaluator. Clearing them is required both at setup and whenever the goal's item is removed,
because a stale aim or sling flag would let the planner build a plan on a state that no longer
exists.

## `add_item` / `remove_item`

**Contract** — `add_item` dispatches on the item's kind and installs that kind's evaluator and
operator set; items that are neither a weapon nor a thrown object are ignored entirely.
`remove_item` first checks whether the current goal names the departing item and, if so, resets
the shared storage and falls back to the idle goal; then removes every evaluator and operator
belonging to the item.

**Invariants** — the goal must be reset *before* the item's operators are removed. A goal
naming an item whose operators no longer exist is unreachable, and the planner would search
its whole space and fail every update thereafter.

Only weapons and thrown objects get an operator set. Everything else a creature carries —
armour, food, artefacts, ammunition — is handled through other paths and contributes nothing
to this planner, which keeps the search space proportional to the *usable* inventory rather
than to the whole rucksack.

**Notes** — removal works by repeatedly looking up the lowest identifier at or above
(item, property zero) and deleting it while it still belongs to the item. The packing puts all
of one item's identifiers in one contiguous range, so this walks exactly that range. The source
marks it as correct but not optimal; the optimal form erases the whole range in one operation,
which a rebuild with an ordered container gets for free.

## `update`

**Contract** — steps the generic planner. In logging builds it first synchronizes its logging
flag with the global AI-debug flag, so the log can be turned on and off while the game runs.

## `action2string` / `property2string`

**Contract** — in logging builds only, render an identifier as text: the owning entity's name
(or "no items" for the reserved pseudo-entity, and also when the entity cannot be found),
a colon, and the operator or property name. Total over both enumerations.

**Notes** — these exist because a plan is otherwise a list of integers, and debugging a
planner means reading its plans. The mapping is exhaustive and asserts on an unknown value,
which makes it a compile-time-checked inventory of the two enumerations — the closest thing
the source has to a declaration that the two name sets are complete. Two entries collide
(both queue-wait operators render as the same text); harmless, but a rebuild should fix it.
