# src/xrGame/smart_cover_animation_planner.cpp

> Builds the operator set that defines everything a creature can do inside a smart cover, and owns the in-cover lifecycle around it.

**Needs** — [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) · [`smart_cover_evaluators.h`](smart_cover_evaluators.h.md) · [`smart_cover_planner_actions.h`](smart_cover_planner_actions.h.md) · [`smart_cover_loophole_planner_actions.h`](smart_cover_loophole_planner_actions.h.md) · [`smart_cover_planner_target_selector.h`](smart_cover_planner_target_selector.h.md) · [`smart_cover_animation_selector.h`](smart_cover_animation_selector.h.md) · [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`property_storage.h`](property_storage.h.md) · [`Hit.h`](Hit.h.md) · [`game_object_space.h`](game_object_space.h.md)
**Used by** — reached through its declarations in [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md); callers name that, not this file.
**Tier floor** — T2: a plan search per cycle over a fixed operator set

## Purpose

The operator table in this file *is* smart-cover behaviour. Nothing else decides what a
creature does in cover: the loopholes supply geometry and clips, the evaluators supply
questions, and this table says which action is possible under which answers and what it
changes. Everything visible — that a creature ducks, pops up, fires, drops back, reloads
when dry, changes loophole when the enemy moves, and leaves with a vault animation when
there is one — is a consequence of these fifteen operators and their preconditions.

The rest of the file is the lifecycle that brackets the plan.

## State

Declared in [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md).

## `setup`

**Contract** — attaches the planner to a creature, registers every evaluator and operator,
and hands itself and the creature's world state to the movement manager's target selector
([`smart_cover_planner_target_selector.h`](smart_cover_planner_target_selector.h.md)).
Called once when the creature's planner set is built, not on every entry into a cover.

**Invariants** — the target selector must receive the planner *and* the outer world state,
not just the planner: it decides which cover and loophole to aim for by reading properties
the outer planner owns.

## `initialize` — entering the cover

**Contract** — hooks a hit callback onto the creature, saves the creature's head turn speed,
seeds five world-state properties, and picks a default goal if the outer layer has not
already set one.

```text
FUNCTION initialize()
  install hit_callback on the creature
  saved_head_speed = creature.head.speed
  world state:
    looked_out             = false
    ready_to_idle          = true
    ready_to_lookout       = false
    ready_to_fire          = false
    ready_to_fire_no_lookout = false
  IF the goal is already set  RETURN
  goal = looked_out
```

**Invariants** —

- The creature enters **idle-ready and nothing else**. The four other readiness properties
  are mutually exclusive with idle-ready in the operator table below, and starting with
  more than one set would let the planner skip a transition.
- The head speed is saved here and restored in `finalize`, so the cover may retune it.
  The original's retune — pinning the head to an eighth-turn per second — is present but
  disabled; what survives is the requirement that any change be *saved and restored*,
  because the creature keeps its speed after leaving.
- The default goal is "have looked out". That makes a creature with no instruction peek
  rather than hide, which is what makes a cover look occupied.

## `finalize` — leaving the cover

**Contract** — finalizes the base planner, finalizes the target selector if it was
initialized, restores the creature's head turn speed, clears the initialized flag, and
removes the hit callback.

**Invariants** — the order matters at one point: the target selector is finalized *after*
the base planner, because the base planner's finalization may run the current action's own
finalization, which still reads the selector's target.

## `target`

**Contract** — sets the goal to a single world property that must become true, replacing
any previous goal. The goal is always exactly one property, never a conjunction — the
smart-cover plans are short enough that a conjunctive goal is never needed, and keeping it
single keeps the search bounded.

## `add_evaluators`

**Contract** — registers the world-state questions. Three kinds appear:

- **Real questions** — cover entered, cover actual, loophole actual, loophole exitable, can
  exit with an animation ([`smart_cover_evaluators.cpp`](smart_cover_evaluators.cpp.md)),
  and readiness to kill, which is the creature's own weapon-state evaluator configured with
  a threshold of six.
- **Pinned-false constants** — looked out, exit smart cover, loophole idle, loophole fire,
  loophole fire-without-looking-out. These are the properties the *actions* set as effects;
  pinning the evaluator to false means "this has not happened yet in the world", so the
  planner must plan an action to achieve it every cycle. It is how a *repeating* behaviour
  is expressed in a planner that otherwise stops once its goal holds.
- **Stored properties** — the four readiness flags, read back out of the planner's own
  world state rather than computed. These carry the creature's phase across planning cycles.

**Invariants** — the distinction between the three kinds is the load-bearing idea. A
property backed by a pinned-false constant can never be satisfied by the world, so the plan
always re-executes; a property backed by storage persists; a property backed by a real
evaluator tracks the world. Getting one of them wrong changes a creature from "peeks
forever" to "peeks once".

## `add_actions` — the operator table

**Contract** — registers fifteen operators. Each is listed below as its preconditions and
effects; the implementations live in
[`smart_cover_planner_actions.cpp`](smart_cover_planner_actions.cpp.md) and
[`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md).

```text
# --- getting into position ------------------------------------------------
change loophole (animated)
  needs  cover_entered, NOT loophole_actual, ready_to_idle, can_exit_with_animation
  gives  loophole_actual, loophole_exitable

change loophole (non-animated)
  needs  cover_entered, NOT loophole_actual, ready_to_idle, NOT can_exit_with_animation
  gives  loophole_actual, loophole_exitable

exit cover (non-animated)
  needs  cover_entered, loophole_exitable, NOT can_exit_with_animation, ready_to_idle
  gives  smart_cover_actual

exit cover (animated)
  needs  cover_entered, ready_to_idle, loophole_exitable, can_exit_with_animation
  gives  smart_cover_actual

# --- doing something at the loophole --------------------------------------
idle              needs  smart_cover_actual, cover_entered, loophole_actual,
                         ready_to_kill, NOT loophole_idle, ready_to_idle
                  gives  loophole_idle

lookout           needs  ... ready_to_kill, NOT looked_out, ready_to_lookout
                  gives  looked_out

fire              needs  ... ready_to_kill, NOT loophole_fire, ready_to_fire
                  gives  loophole_fire

fire_no_lookout   needs  ... ready_to_kill, NOT loophole_fire_no_lookout,
                         ready_to_fire_no_lookout
                  gives  loophole_fire_no_lookout

reload            needs  smart_cover_actual, cover_entered, loophole_actual,
                         NOT ready_to_kill, ready_to_idle
                  gives  ready_to_kill

# --- moving between what you are doing ------------------------------------
idle -> lookout   needs  ... ready_to_idle, NOT ready_to_lookout, ready_to_kill
                  gives  ready_to_lookout, NOT ready_to_idle
lookout -> idle   needs  cover_entered, ready_to_lookout, NOT ready_to_idle
                  gives  ready_to_idle, NOT ready_to_lookout
idle -> fire      needs  ... ready_to_idle, NOT ready_to_fire, ready_to_kill
                  gives  ready_to_fire, NOT ready_to_idle
fire -> idle      needs  cover_entered, ready_to_fire, NOT ready_to_idle
                  gives  ready_to_idle, NOT ready_to_fire
idle -> fire_no_lookout / fire_no_lookout -> idle  — the same pair for the
                  no-lookout firing posture
```

**Invariants** —

- **Idle is the hub.** Every posture is reachable only *through* idle: there is no
  lookout-to-fire operator. A creature that is looking out and wants to fire must drop
  back to idle first, which is why the peek-then-shoot rhythm always has a beat in between.
  This is a structural decision about the authored clips — the transitions that exist are
  the ones animators made — and it is the single constraint that shapes the whole table.
- **The readiness flags are a mutual exclusion.** Exactly one is true at a time, enforced
  by every transition operator clearing the one it leaves.
- **Doing something requires being ready to kill**, with one exception: reload, which
  requires *not* being ready and establishes it. That is how an empty weapon interrupts the
  rhythm without any special case.
- **The two ways out of a cover, and the two ways to change loophole, are distinguished
  only by whether an animation exists.** Both pairs have mutually exclusive preconditions
  on the same evaluator, so exactly one is ever applicable. The planner never chooses
  between them; the authored data does. Note that the animated exit is implemented by the
  *same* action type as an animated loophole change — leaving a cover is a transition to
  the reserved outside vertex, exactly as in
  [`smart_cover_detail.cpp`](smart_cover_detail.cpp.md).
- **Leaving and returning to idle do not require the loophole to be actual**, while
  everything else does. A creature whose target loophole has changed under it can still
  finish dropping back to idle and then re-plan, instead of being stuck mid-posture.

**Notes** — the two loophole-change operators produce `loophole_exitable` as an effect even
though that property is also answered by a real evaluator. That is the planner being told
"after you move, you will be somewhere you can leave from" — a claim about the *future*
state that the evaluator would answer about the present. It is what lets the search plan a
move-then-leave sequence in one go.

## `hit_callback`

**Contract** — invoked when the creature is hit while in the cover. Records the time of the
hit, then forwards the hit to the creature's script hit callback with the damage,
direction, the entity responsible and the bone struck. Always reports that it has *not*
consumed the hit, so normal damage processing continues.

```text
FUNCTION hit_callback(hit) -> bool
  time_object_hit = now
  IF the attacker is the player AND AI is set to ignore the player  RETURN false
  IF the creature is dead                                           RETURN false
  invoke the script hit callback with (damage, direction, attacker, bone)
  RETURN false
```

**Invariants** —

- The time stamp is recorded **before** every early return, including the dead case. The
  dwell evaluator that reads it must see a hit that killed the creature as a hit.
- Returning false unconditionally means this callback observes and never vetoes. A rebuild
  must not use it to absorb damage.

**Notes** — the ignore-the-player check exists only in non-shipping builds; it is a
development convenience for walking through a level unnoticed. The dead check guards the
script callback, not the stamp: a dead creature has no meaningful script-visible reaction.
