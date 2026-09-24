# src/xrGame/agent_member_manager_inline.h

> Roster lookups: index-to-bit and bit-to-index, and the one-line predicates that define squad membership tests.

**Needs** — [`agent_member_manager.h`](agent_member_manager.h.md)
**Used by** — [`agent_member_manager.cpp`](agent_member_manager.cpp.md) · [`agent_member_manager.h`](agent_member_manager.h.md)
**Tier floor** — T3: list search and a bit shift.

## Purpose

The half of the roster that is pure lookup. It is separate from the implementation file
only because the original language wants inline definitions after the class body, but one
decision genuinely lives here: the mapping between a member's position in the roster and
its bit in the squad mask, in both directions.

## Construction

**Contract** — A roster starts empty, with an empty combat mask, a cache declared current
(trivially: both lists are empty), no recorded throw and a zero throw interval — meaning a
freshly built squad may throw immediately, and the interval is configured afterwards.

## `member`

**Contract** — Two lookups. By stalker reference: finds the roster slot whose object is
that stalker, asserting it exists. By mask: walks the roster shifting the mask right one
step per member and stops when the mask has been reduced to its lowest bit — the inverse
of the index-to-bit mapping. Passing a mask with more than one bit set, or a bit past the
roster's end, is a programming error and is unreachable by contract.

## `mask`

**Contract** — One bit shifted left by the member's roster index. Asserts membership.

**Invariants** — This is the definition the whole squad system rests on: *bit position is
roster position*. Everything that reorders or shortens the roster must renumber every mask
that was derived from it (see `remove` in the implementation twin).

## `group_behaviour`

**Contract** — True when the roster holds more than one member. A squad of one behaves as
an individual: no coordination, no distribution, no shared chatter discipline.

## Accessors

**Contract** — `members` in mutable and read-only forms, `combat_mask`, and the throw
interval's getter and setter. No logic.

## Membership predicates

**Contract** — Two small predicates used by the searches above: one matching a roster slot
by the identity of its stalker, one matching it by entity identifier. They differ because
a dying object may be reachable only by identifier. A rebuild needs the two comparisons,
not two types.
