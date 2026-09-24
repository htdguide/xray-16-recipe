# src/xrGame/action_base_script.cpp

> Exports the planner operator to the script virtual machine, so that a creature's actions can be written in Lua and planned alongside the engine's own.

**Needs** — [`action_base.h`](action_base.h.md) · [`script_action_wrapper.h`](script_action_wrapper.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration data

## Purpose

This is one of the two exports that make the whole goal-and-plan layer scriptable — the
other being the planner itself. A script subclasses the exported action type, declares its
preconditions and effects as world-state conditions, and overrides the lifecycle; the
engine's planner then searches over script actions and engine actions interchangeably.

## State

`Stateless.`

## `CScriptActionBaseExport::script_register`

**Contract** — registers the action type into the script virtual machine under the name
`action_base`, with no base class, constructible from script in three forms: with no
arguments, with the game object it acts on, and with that object plus a diagnostic name.
Runs once at script-engine bring-up. The name and the whole member list below are frozen
by conformance criterion 10.

Exported surface:

- `object`, `storage` — read-only views of the acting game object and of the shared
  world-state storage.
- `add_precondition`, `add_effect` — declare a world-state condition this action requires
  or establishes.
- `remove_precondition`, `remove_effect` — withdraw one by its condition identifier, so a
  script action can change its own contract at run time.
- `setup`, `initialize`, `execute`, `finalize` — the lifecycle, each exported *twice*: as
  the method a script may call, and as the engine-side entry the planner calls, which
  dispatches into the script override if one exists and into the base otherwise.
- `weight` and `set_weight` — the planner's edge cost, likewise overridable.

**Invariants** — the double registration of every overridable method is the mechanism that
makes a script subclass's override actually run when the *engine* drives the lifecycle.
Without it a script could only be called from script. A rebuild needs the same
virtual-dispatch bridge in whatever form its binding layer offers.

**Notes**

- Registration is attached to a helper type that exists only to own it; the action type
  itself is a template and cannot carry a registration function. That is a language
  artifact.
- The holder policy makes the script the owner of an action it constructs — the planner
  takes a reference, not ownership — which is why a script must keep its actions alive
  for as long as the planner has them.
- The precondition and effect methods are disambiguated between two overloads at
  registration; only the single-condition form is exported, so a script adds conditions
  one at a time.
