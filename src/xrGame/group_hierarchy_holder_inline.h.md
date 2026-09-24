# src/xrGame/group_hierarchy_holder_inline.h

> The group holder's trivial accessors and its initial state.

**Needs** — [`group_hierarchy_holder.h`](group_hierarchy_holder.h.md)
**Used by** — [`group_hierarchy_holder.h`](group_hierarchy_holder.h.md)
**Tier floor** — T2: accessors

## Purpose

Holds the accessors that must be cheap because they run inside per-frame perception loops.
The split is an artifact of wanting them inlined; a rebuild may put them wherever it likes.

## State

`Stateless` — it defines the initial state of [`group_hierarchy_holder.h`](group_hierarchy_holder.h.md)'s record.

## `CGroupHierarchyHolder` (construction)

**Contract** — a group is created **against a squad**, which must exist; everything else
starts empty. Specifically the three shared perception lists and the coordination manager
all start absent, not empty-but-allocated: their existence is the signal that the group has
members, and the registration path depends on that.

**Invariants** — the five externally-written counters are zeroed here so that a group with no
members reads as zero active, alive and standing rather than as uninitialized.

## `agent_manager`, `squad`, `visible_objects`, `sound_objects`, `hit_objects`

**Contract** — each returns its target and asserts it exists. Calling any of them on a group
with no members is a programming error, not a runtime condition.

**Notes** — a separate, non-asserting query for the coordination manager exists alongside the
asserting one, and the registration path uses it: "is there a manager yet" is a legitimate
question, "give me the manager that must be there" is a different one. Keeping the two
distinct is what lets the assertion be meaningful everywhere else.

## `members`

**Contract** — the member list, read-only. Mutating it outside the registration path would
break the correspondence between list emptiness and the allocation of the shared perception.
