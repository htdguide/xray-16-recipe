# src/xrGame/script_game_object_smart_covers.cpp

> The facade's smart-cover surface: choosing a cover and a loophole to occupy, selecting what to do while in it, aiming out of it, and retuning the dwell times that make the behaviour look deliberate.

**Needs** — [`script_game_object.h`](script_game_object.h.md) · [`script_game_object_impl.h`](script_game_object_impl.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`smart_cover.h`](smart_cover.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: guarded delegation only

## Purpose

One of the nine files the [game object facade](script_game_object.h.md) is split across.
Every method is a guarded delegation onto one creature's movement manager, and the
*vocabulary* is the content: this file is the only place the smart-cover model is visible
as a whole from outside the AI layer.

A **smart cover** is an authored piece of the level — a window frame, a sandbag line, a
doorway — carrying named **loopholes**. A loophole is one usable position within the cover,
with its own field of view, its own effective range, and its own set of authored
animations. A creature occupying a loophole is in one of four **targets**: idle (hidden),
lookout (head up, not shooting), fire (up and shooting) and fire-without-lookout (shooting
blind over the top). Moving between those is animated, which is why the behaviour reads as
intentional rather than as a creature snapping between poses.

All of this applies only to stalkers: the downcast is to the stalker type in every method,
and every other entity gets a script error.

## State

`Stateless.`

## The destination cover, and how an order is assembled

**Contract** — a creature is sent into cover by writing its *target parameters* one field
at a time, not by one call:

```text
set_dest_smart_cover(cover_id)   # or with no argument: clear it, meaning "no cover"
set_dest_loophole(loophole_id)   # or with no argument: clear it, meaning "any loophole"
set_smart_cover_target(position) # aim at a fixed place
set_smart_cover_target(object)   # aim at an entity, tracking it
set_smart_cover_target()         # aim at nothing
```

`get_dest_smart_cover` and `get_dest_smart_cover_name` read back the resolved cover point
and the cover's authored identifier.

**Invariants**

- The parameters are a *destination*, not a command. The creature's movement manager
  reaches them when it can; `movement_target_reached` answers whether the current
  parameters have caught up with the destination, and is the only way a script can tell.
- Clearing is spelled as the **same method with no argument**, writing an empty identifier.
  Empty is therefore a meaningful value throughout — "no cover" and "any loophole" — and a
  rebuild must not conflate it with an unset field.
- The aim target is either a position **or** an object, never both: setting one supersedes
  the other. Aiming at an object tracks it as it moves, which is what makes a creature in a
  window keep a bead on a running player.

## Target selection

**Contract** — five methods put the occupying creature into one of the four targets, plus
one that hands it back to its own judgement:

```text
set_smart_cover_target_idle()
set_smart_cover_target_lookout()
set_smart_cover_target_fire()
set_smart_cover_target_fire_no_lookout()
set_smart_cover_target_default(enabled)   # true: the creature chooses for itself again
```

**Invariants** — all five refuse on a **dead** creature, with a distinct message from the
wrong-type one. This is the only group in the nine facade files that guards on liveness,
and it is load-bearing: a corpse in a loophole still has a movement manager, and driving it
would start an animation on a ragdoll.

`in_smart_cover` answers whether the creature is currently occupying one.

## The target selector callback

**Contract** — `set_smart_cover_target_selector` installs a script function that chooses
the target each time the creature reconsiders, optionally bound to a script object; with no
argument, it clears the selector and returns the choice to the engine.

**Notes**

This is the same three-overload install/bind/clear shape as the rest of the facade's
callbacks — see
[`script_game_object_use.cpp`](script_game_object_use.cpp.md#callbacks). What makes it
worth a separate mention is that it is a *decision* callback rather than a notification:
the engine asks the script what to do, mid-behaviour, and uses the answer. A rebuild that
supports only notification callbacks cannot express it.

## Loophole geometry queries

**Contract** — four predicates asking whether a world position is usable from a loophole:

```text
in_loophole_fov(cover_id, loophole_id, position)   # a named loophole's cone
in_loophole_range(cover_id, loophole_id, position) # a named loophole's range
in_current_loophole_fov(position)                  # the one the creature occupies
in_current_loophole_range(position)                # the one the creature occupies
```

**Invariants** — field of view and range are **separate** tests and both must pass for a
loophole to be usable against a target. Authored covers rely on that: a window loophole may
see a wide arc but only to the far wall, and a sandbag loophole the reverse. A rebuild that
merges them into one "can shoot from here" test changes which covers creatures pick.

The named forms take the cover as well as the loophole because loophole identifiers are
only unique within a cover.

## Dwell times and entry distance

**Contract** — five tuning values, each with a reader and a writer:

- `idle_min_time` / `idle_max_time` — the bounds on how long the creature stays down before
  reconsidering.
- `lookout_min_time` / `lookout_max_time` — the same for the head-up pose.
- `apply_loophole_direction_distance` — how close the creature must be to the cover before
  it starts orienting itself to the loophole's direction rather than to its path.

**Invariants** — the times are bounds on a randomized dwell, not fixed durations. That
randomization is what stops a squad of creatures in one cover popping up in unison, and a
rebuild that uses the minimum or the average produces visibly mechanical behaviour.

The entry distance exists because turning to face a loophole too early makes a creature
sidle toward cover, and too late makes it snap round on arrival. It is a per-creature
tuning of an animation transition, and there is no discoverable reason for any particular
default.

## `use_smart_covers_only`

**Contract** — reader and writer for whether the creature may take ordinary cover positions
at all, or must restrict itself to authored smart covers. Used to keep creatures in a
scripted set-piece from wandering behind scenery the author did not choose.

**Notes**

The failure fallbacks in this file break the facade's usual convention twice over, and both
are worth knowing before copying them:

- every *real*-valued reader falls back to the largest representable number, not to −1.
  A script that does not check will read an absurdly long dwell time rather than an
  obviously invalid one, which fails late instead of loudly.
- `in_smart_cover` falls back to empty text where a yes-or-no answer is expected — an
  artifact of the implementation language that happens to read as *no*. A rebuild simply
  answers no.
