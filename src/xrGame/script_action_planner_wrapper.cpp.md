# src/xrGame/script_action_planner_wrapper.cpp

> Routes a script-defined brain's two points into a script class, and keeps the planner's trace switch in step with the console flag.

**Needs** — [`script_action_planner_wrapper.h`](script_action_planner_wrapper.h.md) · [`script_game_object.h`](script_game_object.h.md) · [`ai_debug.h`](ai_debug.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_action_planner_wrapper.h`](script_action_planner_wrapper.h.md)
**Tier floor** — T2: a call convention across the script boundary, once per decision cycle

## Purpose

The paired-function adapter applied to a planner's two overridable points, plus one thing
that belongs to this file alone: **synchronizing each script planner's tracing with the AI
debug flag, every update.**

## State

`Stateless` in itself; it reads and writes the planner's own trace switch.

## `setup(object)`

**Contract** — seeds the planner's trace switch from the AI debug flag, then invokes the
script object's `setup` with the creature. The script is expected to install its operators
and evaluators there and to call the base form; nothing enforces that.

## `setup_base(planner, object)`

**Contract** — the inherited binding, bypassing the script override, for a script's own
`setup` to build on.

## `update`

**Contract** — reconciles the trace switch with the AI debug flag, then invokes the script
object's `update`. One decision cycle.

```text
FUNCTION update()
  IF planner.tracing != ai_debug_flag_for_script_planners THEN
    planner.set_tracing(ai_debug_flag_for_script_planners)
  script_object.update()
```

**Invariants** — the reconciliation is done **per planner, per update**, and it has to be.
The debug flag is a single global the developer toggles from the console mid-session; script
planners are created continuously as creatures spawn and each holds its own switch; and there
is no notification when the flag changes. Polling every update is what makes toggling the
flag take effect on creatures that already exist. The comparison before the write is what
keeps that polling from being a write per creature per frame.

**Notes** — the whole tracing path is compiled out of a shipping build, along with the trace
switch itself. A rebuild is free to drop it; if it keeps it, the "no notification, so poll
and compare" shape is the part worth copying, because the alternative — a registry of live
planners to notify — buys nothing for a debug feature.

## `update_base(planner)`

**Contract** — the inherited decision cycle: solve, switch the running action if the plan's
first step changed, execute it once. A script's `update` that does not call this does not
think.
