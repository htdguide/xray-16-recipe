# src/xrGame/member_order.h

> What the squad has decided about one of its members this cycle: his cover, his target, his share of the enemy list, and the two events he has been told to react to.

**Needs** — [`member_order_inline.h`](member_order_inline.h.md) · [`agent_manager_space.h`](agent_manager_space.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`xrAICore/Navigation/graph_engine_space.h`](../xrAICore/Navigation/graph_engine_space.h.md) · [`xrAICore/Components/condition_state.h`](../xrAICore/Components/condition_state.h.md)
**Used by** — [`agent_corpse_manager.cpp`](agent_corpse_manager.cpp.md) · [`agent_enemy_manager.cpp`](agent_enemy_manager.cpp.md) · [`agent_explosive_manager.cpp`](agent_explosive_manager.cpp.md) · [`agent_member_manager.cpp`](agent_member_manager.cpp.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`member_order_inline.h`](member_order_inline.h.md)
**Tier floor** — T3: a record

## Purpose

The squad coordinator is a separate mind from the individual stalkers. It runs once per cycle,
looks at everything every member knows, and writes a decision per member; each member then
reads its own decision and plans around it. This record is that decision — the *order* the
squad has given one member — and it is the entire interface between the two levels of the AI.

## State

```text
RECORD MemberOrder
  object            : Stalker         # the member this order is for
  initialized       : bool = true
  probability       : real = 1.0      # the squad's confidence in this member's picture
  enemies           : list<int>       # this member's share of the enemy list, by index
  processed         : bool            # has the coordinator handled him this cycle
  selected_enemy    : int             # the enemy he has been told to engage
  cover             : optional<CoverPoint>   # where he has been told to stand
  detour            : bool            # go around rather than straight at the target
  member_death_reaction : { member, time, processing }
  grenade_reaction      : { grenade, thrower, time, processing }
```

**Invariant** — the cover assignment is *mutable even through a const view*. The coordinator
reassigns cover while iterating a read-only view of its member list, because cover
distribution is a settlement across all members at once and cannot be expressed as a sequence
of independent writes. The mutability is incidental C++; what a rebuild must preserve is that
cover is assigned as a **joint** decision over the whole squad, not per member.

**Invariant** — the `processed` flag is per cycle and is cleared by the coordinator at the
start of each pass. It is how a multi-pass distribution avoids handling a member twice.

**Invariant** — the enemy share is a list of *indices* into the squad's shared enemy list, not
of enemies. The shared list is re-sorted by confidence each cycle, so these indices are only
valid within the cycle that produced them.

**Invariant** — both reaction records carry a `processing` flag distinct from being non-empty.
An assigned reaction that has not started yet can still be withdrawn or reassigned; one that
is being acted on may not, because the member is already walking or speaking. That two-stage
shape is what stops a squad from reassigning a reaction out from under the member performing
it.

## `CMemberDeathReaction`

**Contract** — the order to react to a squad member's death: whose, when it was assigned, and
whether the reaction has begun. Cleared when consumed or withdrawn.

## `CGrenadeReaction`

**Contract** — the order to react to a thrown grenade: the grenade, the object that threw it,
when, and whether the reaction has begun. The thrower is recorded separately from the grenade
because the reaction is both "get away from that" and "he is over there" — a grenade is a
reliable enemy report.

## `CMemberOrder` accessors

**Contract** — read and write each field of the order. `object` asserts the member is present;
everything else is a plain field access. See
[`member_order_inline.h`](member_order_inline.h.md).

**Notes** — the `initialized` flag is set true at construction and never written again, so it
is always true. It is a vestige of a two-phase construction that no longer exists; a rebuild
drops it.
