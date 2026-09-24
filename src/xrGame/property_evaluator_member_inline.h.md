# src/xrGame/property_evaluator_member_inline.h

> Implementations of the comparing evaluator.

**Needs** — [`property_evaluator_member.h`](property_evaluator_member.h.md)
**Used by** — [`property_evaluator_member.h`](property_evaluator_member.h.md)
**Tier floor** — T3: field assignment and one comparison

## Purpose

Supplies the three bodies declared in
[`property_evaluator_member.h`](property_evaluator_member.h.md), where the contracts are
written. Separate only because C++ templates must be defined in a header.

The one decision that lives here rather than in the declaration: construction pins the
storage, and `setup` prefers that pinned storage over the one the planner supplies. That
precedence is what makes cross-subject questions possible, and it is invisible from the
declaration.
