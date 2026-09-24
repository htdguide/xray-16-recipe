# src/xrGame/agent_member_manager.cpp

> The squad roster: who is in the group, which of them are currently fighting, and the group-wide interlocks (grenade throwing, cover detouring, who may speak) that only make sense across the whole roster.

**Needs** — [`agent_member_manager.h`](agent_member_manager.h.md) · [`agent_member_manager_inline.h`](agent_member_manager_inline.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_memory_manager.h`](agent_memory_manager.h.md) · [`member_order.h`](member_order.h.md) · [`memory_space.h`](memory_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`Explosive.h`](Explosive.h.md) · [`Grenade.h`](Grenade.h.md) · [`sound_player.h`](sound_player.h.md) · [`cover_point.h`](cover_point.h.md)
**Used by** — [`agent_member_manager.h`](agent_member_manager.h.md)
**Tier floor** — T3: list management and a bitmask; the mask width is a fixed-width integer decision, not a layout one.

## Purpose

Every squad-wide decision in the engine is addressed by a **squad mask**: a bit per member,
where the bit's position is the member's index in the roster. A perceived enemy carries the
mask of the members who have seen it; combat membership is a mask; a memory record is
shared by masking. This file owns the roster that gives those bits their meaning, and it
owns the consequences of the roster changing shape.

It is a separate file from the squad manager because the roster's invariants — mask width
caps membership, index order defines bit order, removal renumbers everyone above — are
delicate enough to be worth isolating.

## State

```text
RECORD MemberOrder                 # one roster slot; detail in member_order
  object              : reference to a stalker
  cover               : optional<cover point>       # the position it has claimed
  detour              : bool                        # taking the long way round
  grenade_reaction    : { grenade, thrower }        # what it is fleeing, and who threw it
  member_death_reaction : { member }                # whose death it is reacting to

RECORD AgentMemberManager
  manager        : reference to the owning squad manager
  members        : list<MemberOrder>   # order defines each member's mask bit
  combat_members : list<MemberOrder>   # cached subset; see actuality below
  actuality      : bool                # is the cached subset in step with combat_mask?
  combat_mask    : int (fixed width, one bit per member)
  last_throw_time    : int             # global clock of the squad's last grenade throw
  throw_time_interval: int             # minimum milliseconds between squad grenades

# Invariant: members.size() < bit width of the mask type. A squad that would exceed it
#   is a data error, reported with the offending team/squad/group triple.
# Invariant: a member appears in the roster at most once.
# Invariant: combat_mask has no bit set outside the roster's range.
# Invariant: combat_members equals the members whose bit is in combat_mask, whenever
#   actuality holds. Every mutation of combat_mask that could change the subset clears
#   actuality.
```

**Notes** — The mask width is the hard cap on squad size, and it is the one place where a
width is load-bearing in this file: the same mask type is stored in every perception
record across the AI layer, so widening it widens those records too.

## `add`

**Contract** — Admits a member. Ignores anything that is not a living stalker — a corpse
or a non-stalker entity is silently not a squad member, which is how the roster stays
clean without the caller filtering. Appends to the end of the roster, which assigns it the
next bit. Fails an assertion if the roster is already at the mask's width or if the
stalker is already present.

**Invariants** — Appending, never inserting, is what keeps existing members' bits stable.

## `remove`

**Contract** — Removes a member, in a strict order.

```text
FUNCTION remove(member)
  IF member is not a stalker THEN RETURN
  IF registered_in_combat(member)
    unregister_in_combat(member)          # 1. drop it out of the combat mask first
  m = mask(member)
  memory.update_memory_masks(m)           # 2. renumber every shared memory record
  memory.update_memory_mask(m, combat_mask) # 3. renumber the combat mask itself
  erase member from roster                # 4. only now does the index order change
```

**Invariants** — The order is the whole contract. The mask must be computed before the
erase (it is derived from the index), the shared memory records must be renumbered before
the roster shrinks, and the combat mask is renumbered by the same rule as the records so
that it keeps meaning the same members. Doing the erase first loses the bit position and
silently corrupts every mask in the squad's memory.

## `register_in_combat` / `unregister_in_combat` / `registered_in_combat`

**Contract** — Set, clear and test a member's bit in the combat mask. Setting or clearing
invalidates the cached combat subset unless the mask is unchanged — the actuality flag is
conjoined with "the mask would not change", so a redundant register costs nothing.

**Notes** — A commented-out guard would have restricted combat registration to members with
group behaviour (a squad of one does not coordinate). It is disabled, meaning a lone
stalker does register in combat; a rebuild should treat lone-member squads as ordinary.

## `combat_members`

**Contract** — Returns the subset of the roster currently registered in combat,
recomputing it from the combat mask only when the cache is stale. Returns a reference to
the cached list, so the caller must not hold it across anything that can change the mask.

## `non_combat_members_mask`

**Contract** — The complement of the combat mask restricted to actual members: a mask of
everyone on the roster who is *not* fighting. Computed by scanning rather than by
complementing, because the complement of the mask would include bits past the end of the
roster.

## `mask`

**Contract** — Two forms: by stalker reference and by entity identifier. Both find the
member's index in the roster and return one bit shifted to that index. Asserts the member
is present — asking for the mask of a non-member is a bug, not a miss. `get_member` is the
forgiving variant: it looks a member up by entity identifier and answers *absent* rather
than asserting.

## `in_detour` / `can_detour` / `cover_detouring`

**Contract** — `in_detour` counts members currently taking a detour toward cover;
`cover_detouring` answers whether any is; `can_detour` decides whether one more may start.

```text
FUNCTION can_detour() -> bool
  n = in_detour()
  RETURN n == 0 OR n < members.size() / 2
```

**Notes** — The rule is "either nobody is detouring, or fewer than half the squad is". The
first clause is not redundant with the second: for a squad of one or two, half rounds down
to zero or one and the second clause alone would forbid the first detour. The behaviour
bought is that a squad never sends most of itself the long way round at once — somebody
keeps pressure on directly.

## `can_cry_noninfo_phrase`

**Contract** — True when no combat member is currently playing a non-forced sound.
Non-informational combat chatter ("come on!", "over here!") is the lowest-priority speech
in the game and must never step on a member who is already saying something; a
squad-global check is the cheapest way to enforce one voice at a time across the group.

## `can_throw_grenade`

**Contract** — Decides whether the squad may throw a grenade at a location. Refuses if the
squad's throw interval has not elapsed since the last throw, if any member stands within
five metres of the target, or if any member's claimed cover point is within five metres of
it. Read-only.

```text
FUNCTION can_throw_grenade(location) -> bool
  IF now <= last_throw_time + throw_time_interval THEN RETURN false
  FOR EACH m IN members
    IF distance(m.object.position, location) <= 5 THEN RETURN false
    IF m.cover IS PRESENT AND distance(m.cover.position, location) <= 5
      RETURN false
  RETURN true
```

**Notes** — The two radii are separately named but both five metres; they are separate
because they answer different questions — "would this kill him where he stands" and "would
this deny him the place he is heading for". The throw interval is not a constant here: it
is set from outside, per squad, from configuration.

## `on_throw_completed`

**Contract** — Stamps the global clock as the squad's last throw time, starting the
interval. Called when a throw actually leaves a hand, not when one is decided, so a
cancelled throw does not consume the squad's grenade budget.

## `remove_links`

**Contract** — Given a dying object, clears every roster slot's reaction that names it.
Three distinct references must be scrubbed per member: the grenade being fled, the *thrower*
of that grenade, and the fallen member being reacted to. The thrower case is indirect — if
the dying object is the grenade's current parent, the whole reaction is dropped, because a
grenade whose thrower has vanished is no longer attributable and the reaction's purpose was
to blame someone.

## `update`

**Contract** — Nothing. The roster has no per-cycle work; the method exists so the squad
manager can drive all seven subordinates uniformly.
