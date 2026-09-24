# src/xrGame/squad_hierarchy_holder.cpp

> One squad's slot table of groups, created on first mention so that the three-level team/squad/group hierarchy costs nothing for the combinations a level never uses.

**Needs** — [`squad_hierarchy_holder.h`](squad_hierarchy_holder.h.md) · [`squad_hierarchy_holder_inline.h`](squad_hierarchy_holder_inline.h.md) · [`group_hierarchy_holder.h`](group_hierarchy_holder.h.md) · [`seniority_hierarchy_space.h`](seniority_hierarchy_space.h.md)
**Used by** — reached through its declarations in [`squad_hierarchy_holder.h`](squad_hierarchy_holder.h.md); callers name that, not this file.
**Tier floor** — T3: a fixed-size slot table with lazy fill

## Purpose

Creatures are addressed by a three-part identity — team, squad, group — and the shared
knowledge a set of creatures pools (who has seen what, who is fighting whom) hangs off nodes
of that hierarchy. This is the middle node. Its only job is to own the group nodes beneath
it and to name its parent.

The identity triple is authored per creature in its spawn record, and the hierarchy is built
from whichever combinations actually appear — which is why the nodes are created on demand
rather than enumerated up front.

## State

```text
RECORD SquadHierarchyHolder
  team   : reference to the parent team node    # never absent
  groups : list<optional<GroupHierarchyHolder>> # fixed length; entries filled on demand

CONSTANT max_group_count = 32
```

**Invariants** — the slot table is allocated at its full fixed length with every entry
empty, so a group identifier indexes directly and the table never moves. The fixed length is
what makes the identifier a plain index rather than a key; a rebuild that uses a growable
map must keep identifiers stable across insertion, because creatures hold them.

Thirty-two groups per squad is a cap on the authored data, not a simulation limit. An
identifier at or above it is a hard failure reporting the offending value, because it can
only come from a bad spawn record.

## `group`

**Contract** — return the group node for an identifier, creating it on first ask. Const in
intent though it fills a slot. Hard-fails on an out-of-range identifier, naming the value.

```text
FUNCTION group(group_id) -> GroupHierarchyHolder
  REQUIRE group_id < max_group_count
  IF groups[group_id] IS empty
    groups[group_id] = new GroupHierarchyHolder(parent = self)
  RETURN groups[group_id]
```

**Notes** — lazy creation is the whole design. A level authored with team 2, squad 1, group 3
would otherwise allocate every team's every squad's thirty-two groups, each carrying shared
perception state, for one creature. As written it allocates three nodes.

The const-with-mutation shape is the same decision as elsewhere in the chapter: filling a
cache slot is not the object changing. A rebuild without that distinction should make the
accessor plainly mutating.

## `team`, `groups`

**Contract** — the parent node, which is never absent, and the raw slot table for the callers
that iterate every existing group.

## Leadership

**Contract** — when compiled in, the squad records a leader creature and can re-derive it by
taking the first leader found among its groups in slot order.

**Notes** — compiled out in the shipped engine. The re-derivation is a scan with no
tie-break: the leader is whichever group with a leader comes first by identifier, which is
an authoring artifact rather than a decision about seniority. That is probably why it was
disabled; leadership in the shipped games is decided by the squad's script, not here. A
rebuild can drop it.

## Destructor

**Contract** — destroys every group node that was created. Slots never filled cost nothing.
