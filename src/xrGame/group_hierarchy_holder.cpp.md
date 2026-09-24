# src/xrGame/group_hierarchy_holder.cpp

> A group of creatures that pool their perception: joining a group redirects a member's sight, hearing and hit memory into lists the whole group reads.

**Needs** — [`group_hierarchy_holder.h`](group_hierarchy_holder.h.md) · [`squad_hierarchy_holder.h`](squad_hierarchy_holder.h.md) · [`Entity.h`](Entity.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`agent_memory_manager.h`](agent_memory_manager.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`sound_memory_manager.h`](sound_memory_manager.h.md) · [`hit_memory_manager.h`](hit_memory_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: ownership and lifetime of shared structures; no device contact

## Purpose

Creatures in the same group are supposed to know what each other knows — one sees you and
the whole group turns. This file is how that is implemented, and the implementation is not a
message bus: the group owns **three shared lists** (seen objects, heard sounds, received
hits) and every member's own perception system is pointed at them. A member writes what it
perceives straight into the group's list and reads the group's list back as if it were its
own memory. There is no propagation step and no latency.

The consequence a rebuild must accept is that group perception is a *lifetime* problem, not
a messaging problem: the lists must exist before any member points at them and must outlive
every member that does. That is the whole content of this file.

Above the group sits the squad; below it, members. Only one creature type — the human
stalker — brings a coordination manager into the group, and the group creates one lazily the
first time such a member joins.

## State

```text
RECORD Group
  squad           : Squad               # the parent tier; required at construction
  members         : list<Entity>        # insertion-ordered; no duplicates
  visible_objects : optional<list<VisibleObject>>   # the three shared perception lists
  sound_objects   : optional<list<SoundObject>>
  hit_objects     : optional<list<HitObject>>
  agent_manager   : optional<AgentManager>          # created on the first stalker member
```

**Invariants** — the central one, and it is enforced by scattered code:

> The three perception lists exist **exactly when the member list is non-empty**, and the
> coordination manager exists **exactly when at least one member is a stalker**.

A group is destroyed only in that state: empty member list, no lists, no manager. The
destructor asserts all five, which is how the invariant is documented in the original.

## `register_member`

**Contract** — adds a member to the group. Four steps in a **fixed order**, and the order is
the load-bearing part of the file:

```text
FUNCTION register_member(member)
  register_in_group(member)          # 1. list allocation must come first
  register_in_squad(member)          # 2. leader bookkeeping (compiled out in the shipped build)
  register_in_agent_manager(member)  # 3. may create the manager and bind it to the lists
  register_in_group_senses(member)   # 4. point this member's senses at the lists
```

Step one must precede three and four because both of those take references to the shared
lists, and the lists are created by step one. Step three must precede four only in the sense
that both read the lists; they are otherwise independent.

### `register_in_group`

**Contract** — appends the member, and **if the group was empty, first creates the three
shared perception lists**. A member may not be registered twice; that is a programming
error, checked rather than tolerated.

**Notes** — the original reserves no capacity for the lists (the reservation is commented
out at 128 entries each). Three lists per group times many groups made the reservation more
expensive than the growth it avoided. A rebuild should not reintroduce it without measuring.

### `register_in_agent_manager`

**Contract** — if the group has no coordination manager **and this member is a stalker**,
creates one and binds it to all three shared perception lists. Then, if a manager exists at
all, adds the member to it.

**Invariants** — the two tests are separate on purpose: a non-stalker member joining a group
that already has a manager *is* added to it, so a mixed group coordinates over all its
members. Only the *creation* is gated on the member being a stalker.

**Notes** — the manager's binding call is the same call three times with three different list
kinds; the manager distinguishes them by the type it is handed. A rebuild with an explicit
kind argument is equivalent and clearer.

### `register_in_group_senses`

**Contract** — if the member has a perception system at all (it is a monster or a stalker —
not, say, a group-registered vehicle), points its sight, hearing and hit memory at the
group's three lists.

**Invariants** — this is the step that makes group perception work. After it, the member's
"what have I seen" query reads the group's list.

### `register_in_squad`

**Contract** — in the shipped build, nothing. Where leaders are compiled in, it claims the
first live member as the group's leader and, if the squad has no leader, as the squad's.

## `unregister_member`

**Contract** — removes a member. The same four steps in the **same order**, which is *not*
the reverse of registration and must not be made so:

```text
FUNCTION unregister_member(member)
  unregister_in_group(member)          # 1. remove from the list — this is what makes the
                                       #    group empty, which step 3 tests
  unregister_in_squad(member)          # 2. leader bookkeeping
  unregister_in_agent_manager(member)  # 3. may destroy the manager AND the shared lists
  unregister_in_group_senses(member)   # 4. clear this member's sense bindings
```

**Invariants** — the ordering trap. Step three frees the shared lists once the member list is
empty, and step four then clears the leaving member's pointers to them. Step four must
therefore not *read* the lists, only clear the bindings — which it does. Reversing the two
steps would be safer but would change nothing observable; reversing one and two would break
the leader update, which needs the leaving member already gone from the list.

### `unregister_in_group`

**Contract** — removes the member from the list. The member must be present; absence is a
programming error.

### `unregister_in_agent_manager`

**Contract** — two independent teardowns, both conditional:

```text
FUNCTION unregister_in_agent_manager(member)
  IF a coordination manager exists THEN
    manager.remove(member)
    IF manager has no members left THEN destroy the manager
  IF the group's member list is now empty THEN
    destroy the three shared perception lists
```

**Invariants** — the manager is destroyed when *its own* member list empties, which is not the
same moment the group empties: a group of one stalker and three dogs loses its manager when
the stalker leaves, while the lists survive until the last dog does. The two lifetimes are
genuinely independent and the original's placement of both in one function is a convenience,
not a coupling.

**Notes** — a subtlety a rebuild must reproduce: the manager holds references to the shared
lists, and here the manager is destroyed **before** the lists. If the destruction order were
reversed the manager would briefly hold dangling references. Any rebuild whose ownership
model makes that impossible is free to order them either way.

### `unregister_in_group_senses`

**Contract** — clears the leaving member's three sense bindings, so it goes back to
perceiving only for itself.

### `unregister_in_squad`

**Contract** — in the shipped build, nothing. Where leaders are compiled in: if the leaving
member was the group's leader, re-elect a group leader, and if it was also the *squad's*
leader, hand the squad either the new group leader or, when the group has none, the duty of
electing one itself.

## `update_leader`

**Contract** — the group's leader is the **first live member in registration order**. Not the
strongest, not the highest-ranking — the oldest survivor. Compiled in only where leaders are
enabled.

**Notes** — leader election is disabled in the shipped build, which means the seniority
hierarchy ships with two of its three tiers doing real work and the leader concept inert. A
rebuild can omit it; what it cannot omit is the squad reference itself, which other code
walks.

## Destruction

**Contract** — a group may only be destroyed empty: no members, no perception lists, no
coordination manager. There is no cleanup here, only the assertion of that state.

**Invariants** — this is deliberate. A group that still holds members at destruction has
members whose sense bindings point into freed memory, and cleaning up here would hide the
real bug — a creature destroyed without being unregistered.
