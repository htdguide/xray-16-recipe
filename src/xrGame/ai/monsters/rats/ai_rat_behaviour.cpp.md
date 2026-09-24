# src/xrGame/ai/monsters/rats/ai_rat_behaviour.cpp

> The rat's think step — three calls — and the rule that makes a nest drift across the world graph over minutes rather than sitting on its spawn point forever.

**Needs** — [`ai_rat.h`](ai_rat.h.md) · [`ai_rat_impl.h`](ai_rat_impl.h.md) · [`../../../rat_state_manager.h`](../../../rat_state_manager.h.md) · [`../../../../xrAICore/Navigation/game_graph.h`](../../../../xrAICore/Navigation/game_graph.h.md) · [`../../../../xrAICore/Navigation/game_level_cross_table.h`](../../../../xrAICore/Navigation/game_level_cross_table.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a clock check and a coarse-graph step

## Purpose

Two routines. One is the entire live think step; the other is the nest migration, which is the
only piece of the rat's behaviour that operates on the cross-level graph rather than inside the
loaded level.

This file supersedes [`ai_rat_fsm.cpp`](ai_rat_fsm.cpp.md), which defines a think step of the
same name and is excluded from the build precisely because two definitions cannot coexist.

## `Think`

**Contract** — one tick of the rat's brain. Called from the creature's scheduled update.

```text
FUNCTION Think()
  update_morale()          # decay or recovery toward the state-appropriate target
  update_home_position()   # follow the leader's anchor; migrate if due
  brain.update()           # run the top of the state stack
```

**Notes** — the ordering is the contract: morale first, because several states test it as their
very first condition; the anchor second, because "how far am I from home" is the other thing
states test; the brain last, with both already settled. A rebuild that runs the brain first
gets states deciding on last tick's numbers.

Compare the dead version in [`ai_rat_fsm.cpp`](ai_rat_fsm.cpp.md), which looped until a state
declared itself settled. The live one runs each state exactly once per tick and lets the state
push or pop; that is the change of brain shape described in [`ai_rat.h`](ai_rat.h.md).

## `update_home_position`

**Contract** — keeps this rat's home anchor equal to its squad leader's, and advances the
leader's anchor along the coarse cross-level graph when its timer expires. Does nothing for a
dead rat.

```text
FUNCTION update_home_position()
  IF dead  RETURN

  leader = my squad's leader
  IF leader IS NOT me
    IF my anchor differs from the leader's
      join_activity_budget(forced: true)    # the nest has moved; wake up and go
    my anchor = leader's anchor

  IF now < graph_point_change_at            RETURN    # not yet due
  IF my coarse vertex IS NOT next_graph_point  RETURN # haven't arrived at the last target

  next_graph_point = my coarse vertex       # ... which is already true; see Notes
  select_next_home_position()               # picks a neighbouring coarse vertex and re-arms the clock
  anchor = world position of next_graph_point
```

**Invariants**

- **Only the leader migrates; the followers copy.** Followers reach the guard above and are
  overwritten by the leader's anchor before the clock is even consulted — but the clock check
  runs for them too, and a follower whose coarse vertex happens to match its own stale target
  will also migrate and then be overwritten next tick. Harmless, wasteful, and worth collapsing
  in a rebuild.
- **A nest that is still travelling does not re-roll.** The second guard means the timer only
  fires once the group has actually arrived at the last chosen coarse vertex, so a long journey
  is not interrupted by the next migration becoming due.
- **Discovering the anchor has moved forces the rat active.** That is the nest's
  wake-up: settled rats do not notice a moved anchor by drifting toward it, they are re-admitted
  to the activity budget and walk.

**Notes** — the assignment immediately before the neighbour choice re-assigns
`next_graph_point` to the value the guard above has just proved it already equals. It is a
no-op. What it was *meant* to be is legible — the two-field current/next pair should step
together, and the step is in fact performed inside the neighbour choice — but as written the
line does nothing and can be deleted.

The neighbour choice and the clock re-arming are in
[`ai_rat_templates.cpp`](ai_rat_templates.cpp.md).
