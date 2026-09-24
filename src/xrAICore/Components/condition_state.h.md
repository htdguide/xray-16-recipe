# src/xrAICore/Components/condition_state.h

> Declares the world state — a sorted, hashed set of property/value pairs — whose operations live in [`condition_state_inline.h`](condition_state_inline.h.md).

**Needs** — [`operator_condition.h`](operator_condition.h.md) · [`condition_state_inline.h`](condition_state_inline.h.md)
**Used by** — [`condition_state_inline.h`](condition_state_inline.h.md) · [`operator_abstract.h`](operator_abstract.h.md) · [`problem_solver.h`](problem_solver.h.md) · [`script_world_state_script.cpp`](script_world_state_script.cpp.md) · [`graph_engine_space.h`](../Navigation/graph_engine_space.h.md) · [`member_order.h`](../../xrGame/member_order.h.md)
**Tier floor** — T2: a sorted associative set with an incremental hash. Any language with a sequence type expresses it.

## Purpose

Declares the surface implemented in [`condition_state_inline.h`](condition_state_inline.h.md).
A world state is the planner's *vertex*: a partial description of the world as a set of
property/value pairs. The property type is left open so that the same machinery can serve
the symbolic planner (numeric property identifiers, boolean values) and anything else that
wants a partially specified state.

## Exported units

- `conditions()` — the pairs, in ascending property order; the representation is deliberately public
- `add_condition(pair)` — insert keeping the order, no duplicate property
- `add_condition(position, pair)` — insert at a position the caller already located
- `add_condition_back(pair)` — append, valid only when the pair extends the order
- `remove_condition(property_id)` — drop the pair for that property
- `clear()` — empty state, hash zero
- `includes(other)` — is every pair of `other` present here with the same value
- `weight(other)` — how many properties the two states disagree on
- `property(property_id)` — look one pair up
- `hash_value()` — the order-independent hash of the whole state
- comparison — total order (for sorted containers) and equality (hash-first)
- difference — reduce this state to the pairs the other state contradicts
