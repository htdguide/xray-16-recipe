# src/xrGame/agent_memory_manager.cpp

> The squad's shared perception: three lists of remembered objects — seen, heard, hit by — whose per-record squad masks make one member's sighting the whole squad's knowledge.

**Needs** — [`agent_memory_manager.h`](agent_memory_manager.h.md) · [`agent_memory_manager_inline.h`](agent_memory_manager_inline.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`memory_space.h`](memory_space.h.md) · [`memory_space_impl.h`](memory_space_impl.h.md) · [`entity_alive.h`](entity_alive.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`agent_memory_manager.h`](agent_memory_manager.h.md)
**Tier floor** — T3: list scans and bit manipulation on a fixed-width mask.

## Purpose

A squad fights on shared information: if one member sees an enemy, every member currently
in combat knows where it is. This file implements that sharing, and it implements it by
*mask propagation* rather than by copying records. Each remembered object carries a squad
mask saying which members know it; sharing is one bitwise operation per record, and
un-sharing on a member's departure is a bit deletion.

The three lists themselves are not owned here — they live in the squad's members' shared
perception storage and are pointed at. This manager owns only the sharing policy.

## State

```text
RECORD AgentMemoryManager
  manager  : reference to the owning squad manager
  visibles : reference to list<VisibleObject>   # seen; also carries a per-record
                                                #   "currently visible" mask
  sounds   : reference to list<SoundObject>     # heard
  hits     : reference to list<HitObject>       # took damage from

# Each record in all three lists carries:
#   object identity, last level time it was perceived, remembered parameters
#   (position, direction, level vertex), and squad_mask — who knows it.
# Invariant: every bit set in any squad_mask names a current roster index.
#   Roster removal must delete that bit from every mask (see update_memory_masks).
# Invariant: the three list references are set before the manager is updated; they
#   are installed from outside rather than allocated here.
```

## `update`

**Contract** — One step of sharing: spreads knowledge across the combat members. Cheap and
idempotent.

```text
FUNCTION reset_memory_masks()
  FOR EACH list IN (visibles, sounds, hits)
    FOR EACH record IN list
      IF record.squad_mask AND combat_mask IS NON-EMPTY
        record.squad_mask = record.squad_mask OR combat_mask
```

**Invariants** — The condition is the policy: a record already known to *at least one*
combat member becomes known to *every* combat member. A record known only to non-combat
members is not spread — the member who is not fighting does not brief the squad. Sharing
is one-way and never revokes; a record leaves the shared picture only when it expires or
the squad shrinks.

**Notes** — This runs first in the squad manager's update order, before enemies are ranked
or distributed, because ranking must see the merged picture rather than each member's own.

## `update_memory_masks` (roster renumbering)

**Contract** — Given the departing member's mask bit, deletes that bit from every squad
mask in all three lists, and additionally from the visibility mask carried by seen records.
Called by the roster on removal, before the roster shrinks.

```text
FUNCTION update_memory_mask(bit, current) -> mask
  # Delete one bit from a mask and close the gap: bits above the deleted position
  # shift down one, bits below stay. This preserves the roster's "bit position is
  # roster position" invariant across a removal from the middle.
  high = bits of current strictly above bit
  low  = bits of current strictly below bit
  RETURN (high SHIFTED RIGHT 1) OR low
```

**Invariants** — The seen list carries two masks per record — who knows about this object
at all, and who can see it *right now* — and both must be renumbered. Missing the second
is a silent corruption: a member would inherit another member's line of sight.

## `object_information`

**Contract** — Given an object, reports the most recent squad-wide knowledge of it: the
level time it was last perceived and the position it was at then. Searches all three
lists and keeps the latest. Leaves its outputs untouched when the object appears in none
of them, so the caller must seed them — typically with a zero time — and can detect the
miss that way.

```text
FUNCTION object_information(object) -> (level_time, position)
  IF object in visibles THEN take that record's time and position
  IF object in sounds AND its time is later THEN take it instead
  IF object in hits   AND its time is later THEN take it instead
```

**Notes** — The seen list is consulted unconditionally and the other two only beat it on
time, which is not an ordering preference — the three are compared purely by recency. The
first is written without a comparison only because nothing has been written yet.

## `remove_links`

**Contract** — Nothing. The remembered records hold identifiers and copied parameters, not
references to live objects, so a dying object needs no scrubbing here — which is precisely
why perception is stored as copies.
