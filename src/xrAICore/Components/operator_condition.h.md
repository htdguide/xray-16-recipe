# src/xrAICore/Components/operator_condition.h

> Declares the atom of the planner's world model — one property identifier bound to one value — implemented in [`operator_condition_inline.h`](operator_condition_inline.h.md).

**Needs** — [`operator_condition_inline.h`](operator_condition_inline.h.md)
**Used by** — [`condition_state.h`](condition_state.h.md) · [`condition_state_inline.h`](condition_state_inline.h.md) · [`operator_condition_inline.h`](operator_condition_inline.h.md) · [`script_world_property_script.cpp`](script_world_property_script.cpp.md) · [`graph_engine_space.h`](../Navigation/graph_engine_space.h.md)
**Tier floor** — T2: an immutable pair with a derived hash.

## Purpose

Declares the surface implemented in
[`operator_condition_inline.h`](operator_condition_inline.h.md). Every world state,
precondition and effect in the planner is a set of these; nothing smaller exists in the
world model.

The property identifier and the value type are both left open. The planner instantiates it
with a numeric property identifier and a boolean value, which is the decision that makes the
whole planning layer propositional.

## Exported units

- construction from a property identifier and a value — the only way to make one; there is
  no mutation afterwards
- `condition()` — the property identifier
- `value()` — the value
- `hash_value()` — the hash derived from both, computed once at construction
- ordering — property first, value second
- equality — both parts equal
