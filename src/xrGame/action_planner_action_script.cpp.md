# src/xrGame/action_planner_action_script.cpp

> Exports the composite planner-action to the script virtual machine, so that a nested brain can be written in Lua.

**Needs** — [`action_planner_action.h`](action_planner_action.h.md) · [`script_action_planner_action_wrapper.h`](script_action_planner_action_wrapper.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

Completes the scriptable goal-and-plan layer: with the leaf action and the planner already
exported, this exports the type that is both, letting a script build a behaviour hierarchy
entirely in Lua and install it in a creature whose top-level brain is the engine's.

## State

`Stateless.`

## `CScriptActionPlannerActionExport::script_register`

**Contract** — registers the composite into the script virtual machine under the name
`planner_action`, declaring *both* of its bases — the exported planner and the exported
action — so that a script instance is usable wherever either is expected. Constructible
from script with no arguments, with the game object it acts on, or with that object plus a
diagnostic name. Exports the four lifecycle methods and the cost function, each in the
double form that lets a script override actually run when the engine drives it. Runs once
at script-engine bring-up; names are frozen by conformance criterion 10.

**Invariants** — this registration must run *after* both base registrations, and declares
that dependency rather than relying on file order. Registering a derived type before its
bases leaves the script layer unable to resolve the inheritance, which surfaces as
mysteriously missing methods on script subclasses rather than as an error.

**Notes** — the precondition and effect methods are not re-exported here; scripts reach
them through the inherited action surface.
