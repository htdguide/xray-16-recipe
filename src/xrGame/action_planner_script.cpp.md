# src/xrGame/action_planner_script.cpp

> Exports the brain to the script virtual machine: build a planner in Lua, give it actions and evaluators, aim it at a goal, and drive it.

**Needs** — [`action_planner.h`](action_planner.h.md) · [`action_base.h`](action_base.h.md) · [`script_action_planner_wrapper.h`](script_action_planner_wrapper.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data plus three adapter functions

## Purpose

The export that makes creature behaviour authorable without touching the engine. Nearly
every non-trivial behaviour in the shipped games is a script-built planner, so this
registration is one of the most load-bearing pieces of the frozen script surface.

## State

`Stateless.`

## `CScriptActionPlannerExport::script_register`

**Contract** — registers the planner into the script virtual machine under the name
`action_planner`, with no base class, default-constructible from script. Runs once at
script-engine bring-up. Names and signatures are frozen by conformance criterion 10.

Exported surface:

- `object`, `storage` — read-only views of the acting game object and of the shared
  world-state storage.
- `setup`, `update` — the lifecycle, each exported in the double form so that a script
  subclass's override runs when the engine drives the brain.
- `add_action` / `remove_action` / `action` — install, withdraw and fetch an action by
  identifier.
- `add_evaluator` / `remove_evaluator` / `evaluator` — the same for evaluators.
- `current_action_id`, `current_action`, `initialized` — what is running.
- `set_goal_world_state` — aim the brain at a world state.
- `actual` — whether the last plan is still valid.
- `clear` — drop every action and evaluator.
- A free function that narrows an action to a planner, for walking a behaviour hierarchy
  from script.

**Invariants** — installing an action or an evaluator **transfers ownership to the
engine**. A script that constructs one and adds it must not keep its own reference alive
expecting to own it; the planner destroys its parts. This is the single most consequential
detail of the export, because the opposite convention (the script owning them) is what a
reader would assume from the rest of the binding layer.

**Notes**

- Three of the exported operations are free functions adapting the engine's surface rather
  than direct method exports: aiming the goal takes the world state *by reference* in the
  engine and by value from script; the validity query is const in the engine and needed
  non-const from script; and the narrowing cast has no method form. In a rebuild whose
  binding layer handles these, all three collapse into ordinary exports.
- The withdrawal methods are disambiguated between overloads at registration; only the
  by-identifier form is exported.
- The debug-only tracing methods (print the brain, print the current and the goal world
  state) are exported when built in. Script written against them fails to load in a
  release build, so shipped scripts do not use them.
