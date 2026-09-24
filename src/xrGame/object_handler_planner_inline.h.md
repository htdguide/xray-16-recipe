# src/xrGame/object_handler_planner_inline.h

> Three accessors: the creature, and the low half of an operator identifier.

**Needs** — [`object_handler_planner.h`](object_handler_planner.h.md)
**Used by** — [`object_handler_planner.h`](object_handler_planner.h.md)
**Tier floor** — T3: field access and a mask

## Purpose

Inline definitions split from the class body by the original language's requirements. Two of
the three belong with the identifier arithmetic in
[`object_handler_planner_impl.h`](object_handler_planner_impl.h.md) and are separated from it
only because they do not need the property enumeration to be visible; the split is arbitrary.

## State

`Stateless.`

## Accessors

**Contract** — `object` hands out the creature the planner is bound to, asserting it exists.

**Contract** — `action_state_id` masks an identifier down to its low sixteen bits, yielding the
property or operator name without the item. `current_action_state_id` applies that to the
operator currently running, which is how the rest of the creature asks "what is it doing with
its hands right now" without caring which item.
