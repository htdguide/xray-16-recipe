# src/xrAICore/Components/script_world_property_script.cpp

> Publishes the planner's world property — one property identifier bound to one value — to the script layer under the name `world_property`.

**Needs** — [`operator_abstract.h`](operator_abstract.h.md) · [`operator_condition.h`](operator_condition.h.md) · [`../Navigation/graph_engine_space.h`](../Navigation/graph_engine_space.h.md) · [Seam: Script binding layer](../../../SYSTEM-REQUIREMENTS.md#seam-script-binding-layer)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a declarative registration of an existing type into the script virtual machine.

## Purpose

Exists only to name the world property in scripts. It is a separate file because the binding
registration must be able to see both the property type and the script layer, and neither the
template that defines the property nor the planner that uses it should have to.

The names chosen here are part of the frozen script surface: the shipped game scripts build
goals out of `world_property(id, value)` literals, so the class name, the constructor arity and
the two accessor names cannot change in a rebuild that intends to load those scripts.

## `world_property` (script class)

**Contract** — registers the planner's property type with:

- a constructor taking a property identifier and a value, in that order
- `condition()` — the property identifier
- `value()` — the value
- ordering and equality, matching the native ones exactly

The registered type is the same object the planner uses; nothing is converted or copied at the
boundary beyond what the binding layer does for value types.

**Notes** — scripts construct these with numeric property identifiers that the game layer's own
headers name. Those numbers are shared between script and engine by convention only — there is
no registry here that would catch a script naming a property the engine does not evaluate. The
failure appears later, as the planner asserting that a property has no evaluator.
