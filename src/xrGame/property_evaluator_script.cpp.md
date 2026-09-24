# src/xrGame/property_evaluator_script.cpp

> Exports the evaluator base class to the script layer, so planner conditions can be written in Lua.

**Needs** — [`property_evaluator.h`](property_evaluator.h.md) · [`property_evaluator_const.h`](property_evaluator_const.h.md) · [`script_property_evaluator_wrapper.h`](script_property_evaluator_wrapper.h.md) · [`script_game_object.h`](script_game_object.h.md) · [Seam: Script binding layer](../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: registration only

## Purpose

Makes the evaluator an *inheritable* type on the script side. Most of the game's AI
conditions ship as Lua classes deriving from this one, so its exported shape is frozen by
conformance criterion 10.

## `script_register`

**Contract** — registers two classes.

`property_evaluator` — the base, registered together with a native **wrapper** that routes
a virtual call into the Lua subclass and provides the *static* counterpart that a Lua
subclass calls to reach the base behaviour:

| Script member | Role |
|---|---|
| `object` (read-only) | the subject **game object** |
| `storage` (read-only) | the shared answer board |
| constructor `()` | evaluator with no subject |
| constructor `(game_object)` | evaluator bound to a subject |
| constructor `(game_object, name)` | as above, plus the planner-log name |
| `setup(object, storage)` | overridable; base version reachable from the subclass |
| `evaluate()` | overridable; base version returns false |

`property_evaluator_const` — the constant evaluator, derived from the base and constructed
from a single boolean.

**Invariants** — the two-argument registration of `setup` and `evaluate` (the virtual and
its static base form) is what makes `base:evaluate(...)` work from Lua. Dropping the static
half breaks every shipped evaluator that chains to its base.

**Notes** — the binding layer is asked to hold these objects with the *default* holder,
meaning the engine owns the lifetime and Lua holds a reference. An evaluator outlives the
Lua call that created it — the planner keeps it — so script-side ownership would collect it
mid-plan.
