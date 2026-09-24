# src/xrGame/ai/monsters/ai_monster_squad.cpp

> The creature pack: collects what its members want, decides what they should each do about it, and arbitrates the resources they would otherwise fight over.

**Needs** — [`ai_monster_squad.h`](ai_monster_squad.h.md) · [`ai_monster_squad_attack.cpp`](ai_monster_squad_attack.cpp.md) · [`ai_monster_squad_rest.cpp`](ai_monster_squad_rest.cpp.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`entity_alive.h`](../../entity_alive.h.md) · [`memory_manager.h`](../../memory_manager.h.md) · [`steering_behaviour.h`](../../steering_behaviour.h.md)
**Used by** — [`ai_monster_squad.h`](ai_monster_squad.h.md)
**Tier floor** — T3: maps keyed by entity, a sort, and some angle arithmetic

## Purpose

A pack of dogs that each independently ran at the player would converge into one point and
block each other. The pack object exists to prevent that, and it does so with the smallest
possible mechanism: members *report* what they are doing, the leader's think runs one
coordination pass over all the reports, and each member is left a *command* it reads on its
next think. No member ever talks to another member.

Two more jobs live here for the same reason. **Claims**: a cover position or a corpse may
be taken by one member at a time, and the pack holds the claim list. **Danger mode**: when
any member hears something dangerous or is hit, the whole pack is put into a heightened
state for a fixed window.

The pack is *not* the alife squad and not the human agent-coordination layer. It is
creature-only, level-local, and addressed by the same team/squad/group triple every entity
carries.

## State

```text
RECORD Pack
  leader          : reference to a member          # the first registered, re-elected on loss
  goals           : map<member, MemberGoal>        # written by members
  commands        : map<member, SquadCommand>      # written by the coordination pass
  claimed_covers  : list<navigation vertex>
  claimed_corpses : list<entity>
  danger_until    : int (ms, global clock)
  DANGER_WINDOW   : int = 8000 ms
```

**Invariants** — every registered member appears in both maps; registration and removal
keep them in step. The leader is a member or nothing. When the last member leaves, both
claim lists are cleared — that is the only place they are bulk-released, so a pack whose
members all die while holding claims is cleaned up by the last removal rather than by any
of them individually.

The claim lists are reserved for twenty covers and ten corpses at construction. Those are
capacity hints, not limits.

## `register_member` / `remove_member`

**Contract** — registration inserts an empty goal and a `none` command, and makes the
member the leader if there is no leader yet. Removal erases both entries, re-elects the
first remaining member as leader if the leader left, and clears the claim lists when the
pack empties.

**Notes** — leadership is *positional*, not earned: the first registered member leads, and
on its death leadership passes to whichever member the goal map iterates first. There is no
strength, rank or proximity test anywhere. A rebuild is free to elect better; nothing else
depends on how the leader was chosen, only on there being exactly one.

Removal returns early if the member is not in the goal map, which means a member present in
the command map but not the goal map would leak. Registration makes that impossible.

## `is_active` / `living_member_count`

**Contract** — the pack coordinates only when it has a leader and **at least two living
members**. A lone survivor is not a pack and gets no commands, which is what lets the last
dog of a pack fall back to purely individual behaviour without a special case. The count
query returns zero rather than one in that situation, deliberately conflating "one member"
with "no pack".

## `report_goal` / `read_command`

**Contract** — a member writes its goal and reads its command; both are plain map lookups
on an entry guaranteed to exist. The asymmetry is the whole design: a member never writes a
command and the pack never writes a goal.

## `inform_about_enemy`

**Contract** — pushes one enemy into **every** member's enemy memory at once. This is the
pack's alarm: one dog that sees the player makes the whole pack aware of him, regardless of
line of sight or distance.

**Notes** — the enemy is added to memory but *not* made visible, and the commented-out call
beside it shows the alternative was considered. The distinction matters: a member that has
the enemy in memory will hunt toward its last known position, while one that had it made
visible would behave as though it could see it. The original chose the weaker, more
defensible propagation.

## `coordinate`

**Contract** — the pack's whole decision, run once per tick **from the leader's think
only**. Clears every command, drops goals whose subject has been destroyed, then runs the
attack coordinator and the idle coordinator in that order. Allocates scratch lists.

```text
FUNCTION coordinate()
  FOR EACH command: command.type = none          # every command is re-derived each pass
  FOR EACH goal WHERE goal.entity is gone: goal.type = none
  coordinate_attack()                            # ai_monster_squad_attack
  coordinate_idle()                              # ai_monster_squad_rest
```

**Invariants** — commands are *not* persistent. Every pass starts from a blank slate, so a
member whose situation no longer matches any coordinator simply has no command and acts
alone. That is what keeps the pack from issuing stale orders after the enemy dies.

The two coordinators run unconditionally and in sequence, and the second can overwrite the
first's commands for a member that appears in both. In practice a member's goal is either an
attack goal or a rest/travel goal, so they do not overlap.

**Notes** — running only on the leader's think means the coordination rate is the
*leader's* scheduler rate, which degrades with distance from the player like everything
else. A distant pack coordinates rarely, which is correct and is why no separate rate exists.

## `drop_references`

**Contract** — called when any object is destroyed. Clears that object out of every goal
and every command in the pack, resetting the type to `none` in both. This is the pack's
half of the engine-wide invariant that a destroyed entity is unreferenced before its memory
is released.

## `count_near`

**Contract** — how many *other* living members are within a radius of a given member.
Feeds density-sensitive behaviour: a creature that finds several packmates already close to
its target backs off.

**Notes** — it measures the distance to the *goal's entity* of each member, not to the
member itself. Since the common goal entity is the shared enemy, this counts packmates
whose *target* is near the asking creature, which is not the same question the name
suggests. Whether that is the intent is not recoverable; it is consistent across both
callers.

## claims: covers and corpses

**Contract** — three operations each: test, claim, release. Both are linear scans over a
short list, which is correct for a pack of a handful of members.

**Invariants** — a claim is advisory. Nothing stops a member ignoring the test, and nothing
releases a claim if the claimant dies without releasing it — except the bulk clear when the
pack empties. In practice a member releases on leaving the state that claimed, which is
where the discipline lives.

## danger mode

**Contract** — any member may put the pack into danger mode, which stays on for eight
seconds of global clock time from the last trigger. Triggered by hearing a dangerous sound
or taking a hit. Read by the home-area behaviour, which becomes far more willing to leave
its territory while the pack is alarmed.

**Notes** — eight seconds is a literal in the class with no derivation. It is the only
pack-wide state with a decay; everything else is recomputed each pass.

## pack grouping steering

**Contract** — an adapter presenting the pack's members to the generic steering layer.
Enumerates the command map, skipping the creature itself, yielding each member's position
as a neighbour; reports the creature's own position each tick. Reports "no neighbours" when
the pack reference is unset, so an unpacked creature steers alone rather than failing.

**Notes** — two module-level tuning scalars for separation strength and radius sit beside
it with their uses commented out. They are a live-tuning leftover; the real factors come
from the constructor.

The enumeration skips the creature itself only at the first position and immediately after
each advance. That is correct, but it means the skip test runs on every step rather than
once, which is a detail of the iterator shape, not a decision.
