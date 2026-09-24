# src/xrGame/script_action_planner_action_wrapper.cpp

> Routes the five overridable points of a nested planner into a script class.

**Needs** — [`script_action_planner_action_wrapper.h`](script_action_planner_action_wrapper.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — [`script_action_planner_action_wrapper.h`](script_action_planner_action_wrapper.h.md)
**Tier floor** — T2: a call convention across the script boundary

## Purpose

The paired-function adapter, applied to the composite operator described in
[`script_action_planner_action_wrapper.h`](script_action_planner_action_wrapper.h.md).
Structurally identical to [`script_action_wrapper.cpp`](script_action_wrapper.cpp.md) with
one deliberate difference, stated under `weight`.

## State

`Stateless.`

## `setup(object, storage)` · `initialize` · `execute` · `finalize`

**Contract** — each invokes the script object's method of the same name. The base forms
invoke the inherited behaviour directly so a script override can reach it without
re-entering itself.

**Invariants** — the inherited behaviour here is the nested planner's own lifecycle: `setup`
binds the sub-planner's world-state storage, `initialize` prepares it to solve, `execute`
runs one of its decision cycles, and `finalize` closes out whatever inner action was running.
A script override that skips the base form does not merely lose a detail — it leaves the
sub-plan inert, because nothing else drives it.

## `weight(from_state, to_state)`

**Contract** — invokes the script object's `weight` and returns its answer **unchecked**.

**Invariants** — this is the one place this file differs from the plain operator adapter,
which clamps a too-low answer up to the number of properties the operator changes and logs it
(see [`script_action_wrapper.cpp`](script_action_wrapper.cpp.md) for why the floor exists).
The composite has no such guard. Nothing about a sub-planner makes the admissibility argument
weaker — a composite still establishes properties in its parent's state space — so this reads
as an omission rather than a decision, and a rebuild should apply the same floor here.

## `weight_base(action, from_state, to_state)`

**Contract** — the inherited cost, bypassing the script override.
