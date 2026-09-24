# src/xrGame/object_handler_space.h

> The planner vocabulary for "how a creature handles the thing in its hands": thirty-eight world properties and thirty-six operators.

**Needs** — _(none)_
**Used by** — [`object_actions.cpp`](object_actions.cpp.md) · [`object_actions.h`](object_actions.h.md) · [`object_actions_inline.h`](object_actions_inline.h.md) · [`object_handler.cpp`](object_handler.cpp.md) · [`object_handler_planner.cpp`](object_handler_planner.cpp.md) · [`object_handler_planner.h`](object_handler_planner.h.md) · [`object_handler_planner_impl.h`](object_handler_planner_impl.h.md) · [`object_handler_planner_missile.cpp`](object_handler_planner_missile.cpp.md) · [`object_handler_planner_weapon.cpp`](object_handler_planner_weapon.cpp.md) · [`stalker_animation_torso.cpp`](stalker_animation_torso.cpp.md)
**Tier floor** — T4: two enumerations

## Purpose

The object-handling planner is a goal-directed search like every other planner in the
engine: a world state is a set of named boolean properties, an operator changes some of
them, and a plan is a path from now to a goal. This file names the properties and the
operators for exactly one planner — the one that decides how a creature draws, stows,
aims, reloads, fires and throws whatever it is holding.

It is its own file so that the planner, its thirty-odd operator classes and the stalker
brain can all name a property without including each other.

## State

`Stateless.`

## World properties

Properties come in three shapes, and the shape is what a rebuild must understand.

```text
ENUM WorldProperty : int (32-bit)

  # --- the item selector ---
  item_id                 # not a boolean: see the encoding note below

  # --- where the item is ---
  hidden                  # nothing is in hand
  shown                   # this item is in hand
  strapped                # slung over the shoulder
  strapped_to_idle        # mid-transition between slung and held
  idle
  idle_strap
  dropped

  # --- per-slot pairs: suffix 1 and 2 are the two weapon slots ---
  switch1        switch2          # the firing-mode toggle has been operated
  aimed1         aimed2           # the aim animation has completed
  aiming1        aiming2          # the aim animation is playing
  aiming_ready1  aiming_ready2    # aimed and ready to fire
  aim_force_full1 aim_force_full2 # aim held all the way in regardless of urgency
  empty1         empty2           # magazine empty
  full1          full2            # magazine full
  ready1         ready2           # able to fire now
  firing1        firing2          # trigger held
  firing_no_reload1               # trigger held, but do not reload when dry
  ammo1          ammo2            # ammunition available to load
  queue_wait1    queue_wait2      # waiting out a burst

  # --- the thrown-object sub-plan ---
  throw_started
  throw_idle
  throw
  threaten                # brandish rather than throw

  # --- the used-object sub-plan (food, medicine, detectors) ---
  prepared
  used
  use_enough

  # --- composite sentinels ---
  no_items       = (all-ones in the high half) | item_id
  no_items_idle  = (all-ones in the high half) | idle
  dummy          = all ones
```

**Invariants** — a property is a 32-bit word whose **low half names the property and whose
high half names the item**. The planner mints one property instance per item by combining
the item's entity identifier with the property name, so "this rifle is loaded" and "that
pistol is loaded" are two distinct properties in one world state. That is why the entity
identifier's 16-bit width appears here: it is exactly the high half.

The two sentinels follow from that encoding. An all-ones item half is the identifier that
no entity can have, so `no_items` and `no_items_idle` are the "no item at all" instances of
the item-selector and idle properties. A rebuild widening the entity identifier must widen
this word to match, or the two halves collide.

The numbered pairs are per weapon *slot*, not per weapon: a creature carries at most two
firearms and the planner reasons about both at once, because choosing which one to bring up
is part of the plan.

## World operators

```text
ENUM WorldOperator : int (32-bit)
  # bringing an item in and out of hand
  show, do_show, hide, drop
  strapping, strapping_to_idle, unstrapping, unstrapping_to_idle, strapped, idle

  # per-slot weapon handling
  aim1, aim2
  aim_force_full1, aim_force_full2
  reload1, reload2
  force_reload1, force_reload2
  fire1, fire_no_reload, fire2
  switch1, switch2
  queue_wait1, queue_wait2
  aiming_ready1, aiming_ready2
  get_ammo1, get_ammo2

  # thrown objects
  throw_start, throw_idle, throw, threaten, after_threaten

  # used objects
  prepare, use

  no_items_idle = (all-ones in the high half) | idle
  dummy         = all ones
```

**Invariants** — operators are keyed per item by the same high/low split, so the identifier
of "reload *this* rifle" differs from "reload *that* rifle". The idle operator gets the
same no-item sentinel as the idle property, because a creature holding nothing still needs
an operator that means "stand there".

**Notes** — `show` and `do_show` are two operators for one outcome. The first is the
planner's abstract "get this into your hands", the second the concrete animation step; the
split lets the planner precondition the abstract one on the item being in a slot, and the
concrete one on the previous item having finished being put away.

`fire_no_reload` has a property but no numbered pair, and `firing_no_reload1` exists only
for the first slot. The mode where a creature fires without topping up is used for the
primary weapon only.

The `strapped` and `idle` *operators* share names with properties of the same names; they
are separate enumerations and the collision is harmless, but a rebuild using one namespace
for both must rename.
