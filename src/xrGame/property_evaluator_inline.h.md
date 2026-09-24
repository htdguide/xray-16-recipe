# src/xrGame/property_evaluator_inline.h

> The default behaviour of an evaluator that overrides nothing.

**Needs** — [`property_evaluator.h`](property_evaluator.h.md)
**Used by** — [`property_evaluator.h`](property_evaluator.h.md)
**Tier floor** — T3: field assignment and one lookup

## Purpose

Supplies the base implementations declared in
[`property_evaluator.h`](property_evaluator.h.md), which is where the contracts are
written. It is a separate file only because C++ templates must be defined in a header;
a rebuild merges it into the declaration.

The decisions worth carrying across:

- **Construction records the subject and the name and leaves the storage unset.** An
  evaluator is legal to build before a planner exists, and illegal to evaluate before one
  binds it.
- **The default answer is `false`.** A question nobody implemented reads as "does not
  hold", so a half-written planner produces no plan rather than a wrong one.
- **The default configuration load does nothing.** Most evaluators need no tuning.
- **Reading another question's answer asserts a storage is bound**, then delegates to it;
  the storage itself fails on an unanswered question.
