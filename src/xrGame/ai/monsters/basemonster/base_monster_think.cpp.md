# src/xrGame/ai/monsters/basemonster/base_monster_think.cpp

> The creature's think: refresh perception, coordinate the pack, run the state machine, translate the result into movement, and report back up to the pack.

**Needs** — [`base_monster.h`](base_monster.h.md) · [`state_manager.h`](../state_manager.h.md) · [`ai_monster_squad.h`](../ai_monster_squad.h.md) · [`ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`control_animation_base.h`](../control_animation_base.h.md) · [`monster_velocity_space.h`](../monster_velocity_space.h.md) · [`detail_path_manager.h`](../../../detail_path_manager.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: a fixed five-step sequence plus two angle comparisons

## Purpose

Eighty lines that are the whole shape of a creature's decision cycle. Everything else in the
chapter is reached from here, in this order, and the order is the design.

## `think`

**Contract** — the creature's decision cycle, run once per scheduled think. Does nothing for
a dead or destroying creature. Allocates nothing itself.

```text
FUNCTION think()
  IF not alive OR being destroyed THEN RETURN

  init_think()                 # the concrete creature's per-think hook; empty by default
  animation.scheduled_init()   # the animation layer's own per-think preparation

  update_memory()              # perception: what do I know now
  pack.update(self)            # coordination: if I am the leader, decide for everyone
  update_state_machine()       # decision, and its consequences
```

**Invariants** — perception is refreshed **before** the pack coordinates, and the pack
coordinates **before** any member's state machine runs. That ordering is what makes the
pack's commands consistent with what its members can actually see, and it works because the
coordination runs only on the leader's think: by the time a follower thinks, the leader's
pass has already happened this frame or an earlier one.

**Notes** — the whole routine is bracketed by profiler scopes in the original. Those are
incidental, but their placement records which three steps were expected to cost anything:
memory, pack, and the state machine.

## `update_state_machine`

**Contract** — runs the state machine, then the three things that must happen after it and
cannot happen inside it.

```text
FUNCTION update_state_machine()
  state_manager.update()          # picks and runs a state; sets an abstract action
  post_state_update()             # derive the run-turn flags from the chosen state
  translate_action_to_path_params()
  notify_pack()                   # report my goal upward
```

**Invariants** — the state machine's only output is an *abstract action* and whatever it
wrote into the control channels. Turning that action into movement parameters is done after,
once, in one place — which is why no state has to know about velocity masks.

## `post_state_update`

**Contract** — decides whether the creature should play a leaning-into-the-turn animation
while running at an enemy. Sets two mutually exclusive flags read by the animation layer.
Does nothing without an enemy.

```text
FUNCTION post_state_update()
  IF no enemy THEN RETURN
  run_turn_left = run_turn_right = false

  IF the current state is an attack state
     AND the creature is moving along a path
     AND the path yields a current direction
  THEN
    IF distance to the enemy > 3 THEN
      path_heading  = heading of the path direction
      enemy_heading = heading toward the enemy
      offset = angular difference between them

      IF offset is between 60 and 150 degrees THEN
        set run_turn_right or run_turn_left, whichever side the enemy is on
```

**Invariants** — the flags are cleared at the top of every call, so they are pure functions
of this tick and never persist.

**Notes** — the two thresholds are the whole gesture. Below 60 degrees the creature is
already running roughly at its enemy and a lean would read as a stagger; above 150 degrees
the enemy is behind it and the lean would be a full turn, which is a different animation
entirely. The three-unit minimum distance stops the flags firing when the creature is
circling its enemy at arm's length, where the angle swings wildly.

This is the mechanism behind the visible behaviour of a dog that keeps its head and body
angled toward you while running past — and it is *not* a decision the state machine makes,
which is why it lives here.

## `notify_pack`

**Contract** — writes the creature's current goal into the pack, derived from its state.
Runs every think, so the pack always sees this tick's intent.

```text
FUNCTION notify_pack()
  goal = empty

  IF the state is an attack state THEN
    goal = { attack_enemy, my current enemy }

  ELSE IF the state is a rest state THEN
    goal.entity = the pack leader
    IF state is idle at rest OR looking at open ground THEN goal.type = rest
    ELSE IF state is walking to a graph point, moving home, moving to a
            restrictor, or walking to cover                THEN goal.type = walk_graph
    ELSE goal.entity = none          # a rest state the pack has no opinion about

  ELSE IF the state is a pack state THEN
    goal = { rest, the pack leader }

  pack.report_goal(self, goal)
```

**Invariants** — a goal with no type is a legal report and means "I am doing something the
pack should not coordinate". A creature in a panic, an eating creature, a creature under
script control all report nothing, and the coordinators simply skip them.

**Notes** — the mapping from state to goal is a switch over concrete state identifiers,
which is the one place in the chapter where the pack layer is coupled to the state
vocabulary. It is the natural place for it — the alternative is every state knowing about
the pack — but it does mean a new rest state must be added here to be coordinated at all.
The default branch clears the entity rather than the type, which leaves a goal with a
`rest` type and no leader; the coordinators tolerate it.
