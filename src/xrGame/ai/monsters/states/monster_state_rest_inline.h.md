# src/xrGame/ai/monsters/states/monster_state_rest_inline.h

> Peacetime: obey a smart terrain's job if you have one, else get inside your permitted volumes,
> else get back to the middle of your territory, else do what the squad says, else alternate idling
> and wandering on a sixty-second clock.

**Needs** — [`monster_state_rest.h`](monster_state_rest.h.md) · [`monster_state_rest_sleep.h`](monster_state_rest_sleep.h.md) · [`monster_state_rest_walk_graph.h`](monster_state_rest_walk_graph.h.md) · [`monster_state_rest_idle.h`](monster_state_rest_idle.h.md) · [`monster_state_rest_fun.h`](monster_state_rest_fun.h.md) · [`monster_state_squad_rest.h`](monster_state_squad_rest.h.md) · [`monster_state_squad_rest_follow.h`](monster_state_squad_rest_follow.h.md) · [`state_move_to_restrictor.h`](state_move_to_restrictor.h.md) · [`monster_state_home_point_rest.h`](monster_state_home_point_rest.h.md) · [`monster_state_smart_terrain_task.h`](monster_state_smart_terrain_task.h.md) · [`../ai_monster_squad.h`](../ai_monster_squad.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`../anomaly_detector.h`](../anomaly_detector.h.md)
**Used by** — [`monster_state_rest.h`](monster_state_rest.h.md)
**Tier floor** — T3: a priority cascade over nine sub-behaviours

## Purpose

The most-run behaviour in the game and the one that connects a creature to the rest of the world's
systems. Its leaf list is a roll-call of everything that can lay a claim on an idle animal — the
alife job system, the restrictor algebra, the territory, the squad — and the cascade in `execute`
is the priority order among them, written out by hand.

It is also the only composite in this directory that **replaces the base machinery entirely**
rather than filling in its hooks, and that distinction is load-bearing. Every other composite
supplies a "choose the next leaf" function that the base calls only when the current leaf reports
itself finished. This one re-runs the whole cascade **every update**, selects, and then executes.
The consequence is that a resting creature is continuously re-evaluated against the world, which is
exactly right for an idle animal in a simulation where smart terrains hand out jobs and restrictors
move; and it means the leaves' own completion tests are never consulted at this level. A rebuild
must reproduce the re-evaluation, not the mechanism.

## State

```text
RECORD RestState
  time_idle_selected : int (milliseconds)  # start of the current idle/wander cycle
  time_last_fun      : int (milliseconds)  # DEAD: zeroed on entry, never read again
```

## `initialize` / `finalize` / `critical_finalize`

**Contract** — entry zeroes the cycle clock — to *now* or to zero, chosen by a coin flip — and
switches on the creature's anomaly detector. Both exits switch the detector off.

**Notes** — the coin flip is a small, effective trick. Setting the clock to zero means the
sixty-second idle window has already expired, so the creature starts by *wandering*; setting it to
now means it starts by *idling*. Half of the creatures that enter peacetime at the same moment —
which is what happens when a level loads, or when a fight ends — therefore do different things.
Without it, a level's entire population would idle in lockstep for sixty seconds and then all start
walking at once.

The anomaly detector is the component that keeps a wandering creature from strolling into an
anomaly. It is on only while resting, because a creature in combat or fleeing is allowed to run
through hazards — see [`../anomaly_detector.h`](../anomaly_detector.h.md).

## `execute` — the priority cascade

**Contract** — every update, walk a fixed priority order; the first claimant wins. A leaf that
claimed the creature on the previous update keeps it until that leaf's own completion test says it
is done, which is how a multi-update job is not interrupted by a lower-priority claim. Then run the
chosen leaf.

```text
FUNCTION execute()
  # "sticky" means: if this leaf ran last update, it keeps the creature
  # until its own completion test fires; otherwise it must pass its start test.
  IF sticky(smart_terrain_job)      select(smart_terrain_job)
  ELSE IF sticky(move_to_restrictor) select(move_to_restrictor)
  ELSE IF sticky(go_home)            select(go_home)
  ELSE IF squad_order == rest        select(squad_rest)
  ELSE IF squad_order == follow      select(squad_follow)
  ELSE
    IF   cycle_start + 60000       > now()   select(idle)
    ELIF cycle_start + 90000       > now()   select(wander_graph_points)
    ELSE cycle_start = now();                select(idle)

  current_leaf.execute()
  previous_leaf = current_leaf
```

**Notes** — the cascade order is the file's real content.

*The alife job outranks everything.* A smart terrain that has handed this creature a task owns it;
nothing else in peacetime may interfere. That is what makes authored places — a camp, a nest, a
patrol — actually populate.

*Restrictor compliance comes second.* A creature standing outside its permitted volumes walks back
into them before doing anything else. The restrictor algebra is subtractive and easy to get
backwards; the rule is in the glossary and the leaf that enforces it is
[`state_move_to_restrictor.h`](state_move_to_restrictor.h.md).

*The territory comes third*, below restrictors, which is the right nesting: a restrictor is an
absolute constraint set by the level author, a home region is a preference.

*Squad orders come fourth* and are the only branch with no sticky test — the squad's order is
re-read every update and switching between resting and following is immediate. The squad leader
issues those orders; see [`../ai_monster_squad.h`](../ai_monster_squad.h.md).

*The default is a sixty-second clock with a ninety-second wrap.* Sixty seconds idling in place,
then thirty seconds wandering between graph points, then the clock resets and it idles again. Both
numbers derive from one constant — the wander window is half the idle window — so the creature
spends two thirds of its undirected peacetime standing around and one third walking. Neither number
is authored; they are the same for every creature in the game.

The stickiness idiom is written out three times in the original, once per leaf, and each instance
reads "if this leaf ran last update and has not completed, it keeps running". A rebuild expresses
it once.

## The two dead leaves

Nine leaves are registered. **Seven can be selected. Two cannot.**

*Sleeping* ([`monster_state_rest_sleep.h`](monster_state_rest_sleep.h.md)) is constructed and
registered and the cascade never names it. Its identifier appears nowhere in this file's body.

*Playing with a corpse* ([`monster_state_rest_fun.h`](monster_state_rest_fun.h.md)) is likewise
registered and never named — and the field that would have gated it, the timestamp of the last play
bout, is zeroed on entry and never read again. Both halves of that behaviour are present and wired
to nothing.

Both leaves are reachable in the **pack** variant of peacetime,
[`../group_states/group_state_rest_inline.h`](../group_states/group_state_rest_inline.h.md), which
does select sleeping. This file is the solitary variant and the two leaves are dead in it. A reader
watching a lone creature in the shipped game will never see either behaviour from this path.

A rebuild has a choice to make and should make it deliberately: implement the cascade as it is and
leave the two leaves out of the solitary variant entirely, or wire them in and change how idle
wildlife behaves across the whole game. This recipe describes them because they are implemented and
because the pack variant uses one of them — not because this file runs them.
