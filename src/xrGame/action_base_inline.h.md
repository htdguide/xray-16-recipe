# src/xrGame/action_base_inline.h

> The planner operator's lifecycle, its hysteresis timer, and its edge cost: what it means for an action to be chosen, run and abandoned.

**Needs** — [`action_base.h`](action_base.h.md) · [`property_storage.h`](property_storage.h.md) · [`Level.h`](Level.h.md) · [`xrEngine/device.h`](../xrEngine/device.h.md)
**Used by** — [`action_base.h`](action_base.h.md)
**Tier floor** — T2: per-update work inside the AI's frame budget; nothing device-facing

## Purpose

The planner searches a graph whose vertices are world states and whose edges are actions.
Winning that search is the planner core's job. What this file decides is everything about
an edge *once it is being traversed*: when the action's body starts and stops running, how
it reads and writes the world state the search reasoned about, how much the edge costs,
and — the load-bearing part — how long the creature is obliged to keep doing it before the
planner is allowed to change its mind.

## State

```text
RECORD Action
  object            : acting object          # the creature this action acts on
  storage           : world-state storage    # shared with the planner and every sibling action
  start_level_time  : int                    # global clock reading at the last initialize
  start_game_time   : int                    # reserved; nothing reads it
  inertia_time      : int                    # minimum milliseconds this action must run once chosen
  weight            : real                   # the planner's cost for traversing this edge
  first_time        : bool                   # true during the first execute after initialize
  action_name       : text                   # diagnostics only
```

Invariants:

- The world-state storage is *not owned*. One storage per creature is shared by its
  planner and all of its actions, which is what makes an action's effect visible to the
  next action's precondition without any plumbing. An action with no storage bound must
  not be asked for a property.
- `weight` is never below the planner core's floor. The clamp is applied on write *and
  again* on every read, because the floor can change and a stored weight would otherwise
  go stale — which is why the cost is read through a function and not a field.
- `start_level_time` is set by `initialize` and by nothing else, so the inertia window is
  measured from the moment the action was chosen, not from the moment it was constructed.

## The lifecycle

**Contract** — four stages, in a fixed order, driven by the planner:

1. **setup** — bind the acting object and the world-state storage, and clear the inertia
   window. Called when the action is installed into a planner, not when it is chosen.
   Requires both bindings to be present.
2. **initialize** — the action has just been chosen. Stamp the start time and raise the
   first-update flag.
3. **execute** — one update while the action is the chosen one. Lower the first-update
   flag. Called repeatedly.
4. **finalize** — the action is being abandoned or has succeeded. Called exactly once per
   initialize.

**Invariants** — initialize and finalize strictly alternate. Debug builds assert this by
raising a flag on initialize and requiring it lowered before the next one, and requiring
it lowered before finalize; the flag is lowered by the first execute. The practical
consequence, and the one a rebuild must reproduce, is that **an action is always executed
at least once between being chosen and being abandoned** — the planner may not choose and
immediately unchoose.

**Notes** — the inertia window is cleared in *setup*, not in initialize. An action that
wants hysteresis therefore sets it from its own initialize or execute; a value set before
the action is installed does not survive.

## `completed`

**Contract** — reports whether the action's obligatory dwell has elapsed: true once the
global clock has passed the start time plus the inertia window. With a zero window it is
true immediately, which is the default.

**Invariants** — this is the hysteresis that keeps the planner from oscillating. Two
actions whose costs are nearly equal, or a world-state property that flickers, would
otherwise make a creature alternate every update and appear to twitch. Setting a non-zero
inertia window on an action forces the creature to commit to it for that long regardless of
what the planner would prefer. The windows are authored per action and are gameplay
tuning, not correctness.

```text
FUNCTION completed() -> bool
  RETURN now >= start_level_time + inertia_time
```

**Notes** — the clock is the engine's global wall clock, not the level's simulated time,
so the window does not stretch under slow frames and does not survive a save.

## `weight`

**Contract** — the planner's cost for taking this edge, given the world state before and
after. The base ignores both states and returns the stored weight, clamped up to the
planner core's minimum. Concrete actions override it to price themselves by circumstance —
distance to the target, ammunition remaining, danger.

**Invariants** — the cost must never fall below the core's floor, because a zero or
negative edge would let the search find a free cycle and never terminate. The clamp inside
the const read is why the stored weight is mutable state: the read is allowed to repair it.

## `set_weight`

**Contract** — sets the stored cost, clamped up to the floor.

## `set_property` / `property`

**Contract** — write and read one named world-state property in the shared storage.
Writing a property is how an action reports what it has achieved; the planner re-plans when
the stored state stops matching the plan's assumptions. Both require a bound storage.

## `first_time`

**Contract** — true during the first update after the action was chosen. Concrete actions
use it to do once-per-selection work — start an animation, pick a target — without needing
their own flag.

## `init`

**Contract** — the constructor's body, separated so that an action can be re-bound to a
different object after construction. Clears the storage binding, sets the cost to one, and
records the name. Does not touch the inertia window or the timers, which are only
meaningful after setup.

## Diagnostics

**Contract** — in debug builds each lifecycle stage may print a timestamped line naming
the action, gated per action instance so that a developer can trace one creature's
decisions without drowning in every creature's. Compiled out entirely otherwise.
