# src/xrGame/member_order_inline.h

> Construction and field access for a squad member's standing order.

**Needs** — [`member_order.h`](member_order.h.md)
**Used by** — [`member_order.h`](member_order.h.md)
**Tier floor** — T3: field reads

## Purpose

Carries the order record's bodies out of the declaration. A rebuild folds them in; the
substance is in [`member_order.h`](member_order.h.md).

## State

`Stateless.`

## Construction

**Contract** — binds the order to its member and starts it neutral: no cover, full confidence,
unprocessed, no selected enemy, no detour. The member must be present.

## Accessors

**Contract** — reader and writer pairs for confidence, the processed flag, the selected enemy,
the cover assignment and the detour flag; readers for the member, the initialized flag and the
two reaction records; and a mutable view of the enemy share so the coordinator can fill it in
place rather than assigning a new list.

**Notes** — the cover writer is callable through a read-only view of the order, which is the
mechanism behind the joint cover assignment described in
[`member_order.h`](member_order.h.md). The reaction accessors likewise hand out mutable views
so the coordinator can set or clear a reaction's fields individually.
