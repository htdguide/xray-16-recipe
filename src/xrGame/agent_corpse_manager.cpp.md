# src/xrGame/agent_corpse_manager.cpp

> Decides which squad member reacts to which fallen comrade, so that a squad losing two people does not have everyone shout about the same body.

**Needs** — [`agent_corpse_manager.h`](agent_corpse_manager.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`member_order.h`](member_order.h.md) · [`member_corpse.h`](member_corpse.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md)
**Used by** — reached through its declarations in [`agent_corpse_manager.h`](agent_corpse_manager.h.md); callers name that, not this file.
**Tier floor** — T2: a small assignment search run on a squad event

## Purpose

An *agent* in this codebase is a squad's shared brain: a layer above the individual
creature planners that owns the facts several members must agree about. This is one of its
four managers, and it answers a single question — **when squad members die, who reacts to
whom**.

The problem it exists to solve is duplication. Every member who can see a body would
otherwise independently decide to react to it, and the squad would play the same barked
line four times. So the reaction is *assigned*: each body gets at most one reactor and
each member reacts to at most one body.

## State

```text
RECORD CorpseEntry
  corpse  : creature       # the fallen squad member
  reactor : optional<creature>   # the member assigned to react; none until matched
  time    : int            # global clock reading at which the death was registered

RECORD CorpseManager
  corpses : list<CorpseEntry>    # pending, unassigned deaths
  squad   : agent                # the squad brain this belongs to
```

Invariants:

- The list holds only *unhandled* deaths. Once an entry is assigned a reactor the reaction
  is handed to that member and the entry is removed, so the list is a work queue and not a
  record of the dead.
- A body may be registered only once; registering twice is a contract violation, checked.
- The registration timestamp is carried through to the reacting member, so that the
  member's own reaction knows how long ago the death happened and can choose a different
  response for a stale one.

## `register_corpse`

**Contract** — records a squad member's death as pending, with no reactor and stamped with
the current clock. Fails on a duplicate.

## `process_corpse`

**Contract** — picks the best pending body for one member and claims it, or reports that
there is nothing for this member. Considers only bodies the member can *see right now* —
not remembers, not could see — because the reaction is a visible, immediate one. Among
those it takes the nearest, subject to one rule.

**Invariants** — the rule is the interesting part: a body that already has a reactor is
skipped **unless this member is strictly closer to it than its current reactor is**. That
is what lets the assignment improve — a member who arrives with a better claim takes the
body, and the displaced member is left to be re-matched on a later pass.

```text
FUNCTION claim_a_corpse(member) -> bool
  best = none ; best_distance = infinity
  FOR EACH entry IN corpses
    IF NOT member.can_see_now(entry.corpse) THEN CONTINUE
    d = squared_distance(entry.corpse, member)
    IF d >= best_distance THEN CONTINUE
    IF entry.reactor exists
       AND squared_distance(entry.reactor, entry.corpse) <= d THEN CONTINUE
       # someone nearer already has it
    best = entry ; best_distance = d

  IF best is none THEN RETURN false
  best.reactor = member
  RETURN true
```

**Notes** — comparing the incumbent's distance against `best_distance` rather than against
this member's own distance means the test tightens as the loop finds nearer candidates.
The effect is the same for the winning entry, since `best_distance` equals this member's
distance at the moment the claim is made; earlier iterations may reject an entry the
member could in principle have taken, which only costs an extra pass.

## `react_on_member_death`

**Contract** — runs the assignment to a fixed point, then hands each assigned body to its
reactor as that member's pending death reaction and clears the assigned entries.

```text
FUNCTION react_on_member_death()
  REPEAT
    changed = false
    FOR EACH member IN squad.combat_members
      IF member is not already processing a death reaction THEN
        changed = claim_a_corpse(member)
    UNTIL NOT changed

  FOR EACH entry IN corpses WITH a reactor
    reaction = squad.member(entry.reactor).death_reaction
    reaction.subject = entry.corpse
    reaction.time    = entry.time
    reaction.processing = true
  remove every entry that has a reactor
```

**Invariants**

- Only members **in combat** are considered. A squad member who is not fighting has other
  things to do and reacting to a death is a combat behaviour.
- A member already processing a death reaction is skipped, so one member never handles two
  deaths at once.
- Iterating to a fixed point is what lets displacement settle: a member bumped off a body
  by a nearer claimant gets another turn.

**Notes** — the loop's continuation flag is *assigned* rather than accumulated across
members, so only the last member examined in a pass decides whether another pass runs. A
pass in which an early member claimed a body and the last one did not therefore terminates
early. The intent is plainly an accumulation, and a rebuild should accumulate; the shipped
behaviour is one or two fewer refinement passes, which in a squad of four is rarely
visible. The identical mistake appears in the explosive manager's matching loop.

## `remove_links`

**Contract** — drops every pending entry naming a given object, called when that object
leaves the simulation. A pending body whose object is gone would otherwise be assigned to
a member who then reacts to nothing.

## `clear`

**Contract** — empties the pending list, for a squad being torn down or reformed.

## `update`

**Contract** — does nothing. The manager is event-driven: assignment happens when a member
dies, not on a schedule. The empty per-frame hook exists so the squad brain can call all
four of its managers uniformly, and a rebuild should let a manager simply not have one.
