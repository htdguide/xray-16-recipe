# src/xrGame/object_handler_planner_weapon.cpp

> The whole firearm handling model as planner data: thirty evaluators and twenty-five operators, installed per weapon a creature picks up.

**Needs** — [`object_handler_planner.h`](object_handler_planner.h.md) · [`object_handler_planner_impl.h`](object_handler_planner_impl.h.md) · [`object_handler_space.h`](object_handler_space.h.md) · [`object_property_evaluators.h`](object_property_evaluators.h.md) · [`object_actions.h`](object_actions.h.md) · [`Weapon.h`](Weapon.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: declarative planner-set construction

## Purpose

This file contains no algorithm. It is a *declaration*: the complete precondition-and-effect
table for using a firearm, written as constructor calls. Read as data, it is the specification
of how the engine believes a rifle is operated — which is exactly what a rebuild needs, and is
written down nowhere else.

Two functions, both called once per weapon the creature acquires and undone when the weapon
leaves.

## State

`Stateless.`

## `add_evaluators` — what the planner can observe

**Contract** — installs one evaluator per property of this weapon. An evaluator answers
"is this property true right now" by looking at the world; the planner's search uses those
answers as its start state. Three kinds appear:

**Observed** — the property is read from the weapon or the creature:

```text
  hidden          # is the weapon out of the creature's hands
  ammo1 / ammo2   # does the creature carry ammunition for this barrel
  empty1 / empty2 # is the barrel's magazine empty
  full1 / full2   # is it at capacity
  ready1 / ready2 # can the barrel fire right now
  queue_wait1/2   # is a burst still in flight on this barrel
```

**Shared-storage** — the property is not derived, it is a flag the operators wrote, so the
evaluator just reads the storage:

```text
  aimed1, aimed2, strapped, strapped_to_idle
```

**Constant** — the evaluator always answers a fixed value:

```text
  switch1  = true      # the first barrel is selected by default
  switch2  = false
  firing1, firing_no_reload1, firing2      = false
  idle, idle_strap, dropped                = false
  aiming1, aiming2                         = false
  aiming_ready1, aiming_ready2             = false
  aim_force_full1, aim_force_full2         = false
```

**Invariants** — the constant-false evaluators are the load-bearing trick of the whole
planner. A property that means *"I performed this action"* must never be observable as already
true, or the planner concludes the goal is met and does nothing. Making its evaluator a
constant false forces the plan to contain the operator that produces it. Every such property
is one whose truth is an *event*, not a *state*.

The two switch evaluators are constants of opposite value, marked in the source as temporary.
They encode "a weapon starts with its primary barrel selected" as an unchanging fact, which
means the *observed* state never reflects a mode change — only the switch operators' effects
do, within one plan. A rebuild wanting a weapon whose fire mode persists across plans must
make these real evaluators.

The barrel-numbered properties exist in pairs for every weapon, including single-barrelled
ones. The second barrel's evaluators simply always answer the empty-weapon answer; the plan
never reaches them.

## `add_operators` — what the planner can do

**Contract** — installs twenty-five operators for this weapon, each with its preconditions and
effects, and finally sets eight inertia times. Below is the table, which is the content of
this file.

A precondition that recurs on almost every operator: **not slung and not mid-sling**. It is
omitted from the list below except where its absence is meaningful.

```text
show          needs: hidden, and no other item selected
              gives: not hidden, this item selected

hide          needs: not hidden, this item selected, not slung, not mid-sling
              gives: hidden, no item selected, both aims cleared

drop          needs: not hidden, not slung, not mid-sling
              gives: dropped, both aims cleared

idle          needs: not hidden, not slung, not mid-sling
              gives: idle, both aims cleared

strapping     needs: not hidden, not slung
              gives: slung, mid-sling, both aims cleared
strapping_to_idle   needs: not hidden, slung, mid-sling
              gives: not mid-sling
unstrapping   needs: not hidden, slung
              gives: not slung, mid-sling
unstrapping_to_idle needs: not hidden, not slung, mid-sling
              gives: not mid-sling
strapped      needs: not hidden, slung, not mid-sling, not already slung-idle
              gives: slung-idle

aim1          needs: not hidden, barrel 1 selected
              gives: aimed1, aiming1, aimed2 cleared
aim2          needs: not hidden, barrel 2 selected
              gives: aimed2, aiming2, aimed1 cleared

aiming_ready1 needs: not hidden, barrel 1 selected, barrel 1 ready
              gives: aimed1, aiming_ready1, aimed2 cleared
aiming_ready2 needs: not hidden, barrel 2 selected
              gives: aimed2, aiming_ready2, aimed1 cleared

aim_force_full1 needs: not hidden, barrel 1 selected, ready1, full1
              gives: aimed1, aim_force_full1, aimed2 cleared
aim_force_full2 needs: not hidden, barrel 2 selected, ready2, full2
              gives: aimed2, aim_force_full2, aimed1 cleared

queue_wait1   needs: not hidden, barrel 1 selected, not already waiting on 1
              gives: queue_wait1, aimed2 cleared
queue_wait2   needs: not hidden, barrel 1 selected, not already waiting on 2
              gives: queue_wait2, aimed1 cleared

fire1         needs: not hidden, ready1, not empty1, aimed1, barrel 1 selected,
                     queue_wait1 satisfied
              gives: firing1
fire2         needs: not hidden, ready2, not empty2, aimed2, barrel 2 selected,
                     queue_wait2 satisfied
              gives: firing2
fire_no_reload  needs: not hidden, barrel 1 selected
              gives: firing_no_reload1

reload1       needs: not hidden, NOT ready1, ammo1 available
              gives: not empty1, ready1, both aims cleared
reload2       needs: not hidden, NOT ready2, ammo2 available
              gives: not empty2, ready2, both aims cleared
force_reload1 needs: not hidden, NOT full1, ammo1 available
              gives: not empty1, ready1, full1, both aims cleared
force_reload2 needs: not hidden, NOT full2, ammo2 available
              gives: not empty2, ready2, full2, both aims cleared

switch1       needs: barrel 1 not selected, barrel 2 selected
              gives: barrel 1 selected, barrel 2 not, both aims cleared
switch2       needs: barrel 1 selected, barrel 2 not
              gives: barrel 2 selected, barrel 1 not, both aims cleared

get_ammo1     needs: not hidden, NOT ammo1
              gives: ammo1                     # a fake operator; see the note
get_ammo2     needs: not hidden, NOT ammo2
              gives: ammo2
```

**Invariants** — the pervasive "not slung, not mid-sling" precondition is what makes the plan
insert an unsling sequence automatically before anything useful. The planner never has to be
told to unsling; every action simply refuses to run while slung, so the search routes through
the unsling operators. This is the single clearest illustration in the codebase of what the
precondition-and-effect representation buys.

**Every operator that changes the weapon's configuration clears both aim flags.** Reloading,
switching mode, dropping, holstering and slinging all do it. A creature must re-aim after any
of them, and that requirement is expressed once per operator rather than as a rule. The
aiming operators are the mirror image: each sets its own aim flag and clears the *other*
barrel's, because one pair of hands cannot be aimed at two things.

The sling cycle is a four-operator ring with the mid-sling flag as its baton:
*strapping* sets both slung and mid-sling, *strapping-to-idle* clears mid-sling;
*unstrapping* clears slung and sets mid-sling, *unstrapping-to-idle* clears mid-sling. The
mid-sling flag being set is what blocks every other operator during the transition, and the
"to idle" halves exist to clear it.

The *reload* pair fires when the barrel is not ready, the *force-reload* pair when it is not
full. So a creature tops up a half-empty magazine only when a plan explicitly wants full,
which is how the engine distinguishes "reload because I am dry" from "reload because there is
a lull".

**Notes** — the two get-ammo operators have no behaviour at all: their action class is the
plain base, and their entire effect is to assert that ammunition has appeared. They are
*fake* operators, and the source says so. Their purpose is to keep the search space connected:
without them a creature with an empty weapon and no ammunition has no path to any firing goal,
and the planner fails rather than producing the reload-and-fire plan it would produce the
moment ammunition arrives. With them, the plan exists and simply stalls at the get-ammo step,
which is a far better failure mode — the creature stands ready instead of falling back to a
default behaviour. A rebuild needs some equivalent, or creatures will visibly give up when dry.

Three details look like editing slips and are worth flagging rather than smoothing over,
because a rebuild copying the table will reproduce them: *queue-wait 2* requires **barrel 1**
selected, not barrel 2; *aiming-ready 2* omits the ready-2 precondition that its barrel-1
counterpart has; and *force-reload 2* is constructed with the barrel index zero rather than
one, so it issues a barrel-1 reload while claiming barrel-2 effects. None of them is
explicable from the surrounding code.

*Fire-no-reload* has had four of its preconditions commented out — not empty, aimed, and the
queue-wait — leaving only "not hidden and barrel 1 selected". It will therefore fire an
unaimed, possibly empty weapon. Given its name, that is plausibly the point (it is the
operator for suppressive or warning fire), but the commented-out lines mean the current
behaviour was reached by deletion rather than design.

The eight inertia times at the end are the minimum durations the planner must let each
operator run: 500 milliseconds for all six aiming operators, 300 for the two queue-waits. The
queue-wait figure is overwritten per goal by the burst schedule (see
[`object_handler_planner.cpp`](object_handler_planner.cpp.md)); the aim figure is overwritten
per creature by the object handler's aim-time setter. These are the defaults a creature uses
before anyone sets either.
