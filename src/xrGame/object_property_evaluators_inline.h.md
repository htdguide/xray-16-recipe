# src/xrGame/object_property_evaluators_inline.h

> The evaluator base's construction and its one accessor.

**Needs** — [`object_property_evaluators.h`](object_property_evaluators.h.md)
**Used by** — [`object_property_evaluators.h`](object_property_evaluators.h.md)
**Tier floor** — T3: field access

## Purpose

The base is a template over the item type, which forces its definitions into a header; the
split is incidental. Nothing here is more than a field assignment.

## State

`Stateless.`

## Construction and access

**Contract** — construction binds the creature and the item the evaluator observes.
`object` hands out the creature, asserting it exists — a construction invariant, so checked
builds only. The no-items evaluator, which holds no item, gets its own identical accessor.

**Notes** — every evaluator holds *both* the item and the creature, even those that read only
one. The creature is needed to reach the inventory, which is where "is this item the one in
hand" is answered; the item is needed for everything else. A rebuild can carry just the
creature and let each evaluator look the item up, at the cost of a lookup per evaluation per
frame — which is the trade this design declines.
