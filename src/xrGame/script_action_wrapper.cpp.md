# src/xrGame/script_action_wrapper.cpp

> Routes a planner operator's five overridable points into a script class, and refuses to let a script quote a cost that would break the search's admissibility.

**Needs** — [`script_action_wrapper.h`](script_action_wrapper.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`ai_space.h`](ai_space.h.md) · [`xrScriptEngine/script_engine.hpp`](../xrScriptEngine/script_engine.hpp.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_action_wrapper.h`](script_action_wrapper.h.md)
**Tier floor** — T2: a call convention across the script boundary, inside the planner's search

## Purpose

The same paired-function adapter pattern as
[`script_property_evaluator_wrapper.cpp`](script_property_evaluator_wrapper.cpp.md), applied
to an operator's five overridable methods — plus the one decision that belongs to this file
alone: what to do when a script's cost function answers too low.

## State

`Stateless.`

## `setup(object, storage)` · `initialize` · `execute` · `finalize`

**Contract** — each invokes the script object's method of the same name, forwarding its
arguments. The script is expected to call the base form from inside its override; nothing
enforces that, and an override that does not will leave the operator's engine-side state
unbound or its enter/leave bracket unbalanced.

## `setup_base` · `initialize_base` · `execute_base` · `finalize_base`

**Contract** — invoke the inherited method directly, bypassing the script override. This is
what a script's own override calls to get the standard behaviour; without the bypass it would
re-enter itself.

## `weight(from_state, to_state)`

**Contract** — invokes the script object's `weight` with the two world states and returns its
answer as this operator's cost for that search edge. **If the answer is below the operator's
floor, it is corrected up to the floor and a script error is logged naming both numbers.**

```text
FUNCTION weight(from_state, to_state) -> int
  cost = script_object.weight(from_state, to_state)
  floor = number of world properties this operator changes between the two states
  IF cost < floor THEN
    log script error "weight is less than effect count, corrected from <cost> to <floor>"
    cost = floor
  RETURN cost
```

**Invariants**

- The floor is **the count of properties the operator changes**, and the reason is the
  search, not taste. The planner searches with a heuristic that counts how many properties
  still differ from the goal, and that heuristic is only admissible — only guaranteed to find
  the cheapest plan — if no single operator can close more properties than it costs. A script
  quoting a cost of one for an operator that establishes three properties makes the heuristic
  overestimate, and the planner starts returning plans that are not the cheapest and,
  depending on the graph, not even valid.
- The correction is silent in effect and loud in the log: the plan still runs, with the
  clamped cost. Failing instead would kill a creature's brain on a modder's arithmetic
  mistake; clamping degrades the plan quality and says so.
- The floor is computed per state pair, not per operator, because how many properties an
  operator changes depends on which state it is applied from.

**Notes** — the correction is reported **every time the search evaluates that edge**, which
during a single planning cycle is many times. That is the same deliberate flood as the
evaluator adapter's error path: a cost mistake is invisible in gameplay and has to be made
impossible to ignore in the log.

## `weight_base(action, from_state, to_state)`

**Contract** — the inherited cost, bypassing the script override. The inherited form returns
the operator's configured constant cost, itself raised to the floor. So a script that simply
delegates to the base gets a correct cost for free, and the correction above only ever fires
for a script computing its own.
