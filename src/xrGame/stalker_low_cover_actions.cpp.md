# src/xrGame/stalker_low_cover_actions.cpp

> Low cover: a wall you can hide behind or shoot over, but not both, so the choice is made every cycle.

**Needs** — [`stalker_low_cover_actions.h`](stalker_low_cover_actions.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`stalker_combat_planner.h`](stalker_combat_planner.h.md) · [`stalker_planner.h`](stalker_planner.h.md) · [`sound_player.h`](sound_player.h.md)
**Used by** — [`stalker_low_cover_actions.h`](stalker_low_cover_actions.h.md)
**Tier floor** — T2: per-cycle posture, aim and cover-hint updates.

## Purpose

*Low cover* is a navigation position whose protection depends on posture: crouched you are
behind it, standing you are exposed but can shoot over it. That makes it the one combat
situation where posture is the tactical decision, and this file is the three postures a
stalker cycles between at such a position.

None of the three actions moves the creature. The planner that owns them pins it in place,
and all three do their work through posture, aim and the cover hint.

## State

Declared but unused: both fighting actions carry a last-changed timestamp and a private
random source that nothing reads. The intent is legible — alternate between crouched and
standing on a randomized interval, the way the look-out action in the danger branch does —
and it was not implemented. A rebuild should either implement it or drop the fields; what
ships is a creature that stands the whole time it is shooting.

## The cover hint

**Contract** — all three actions, and the planner's own per-cycle hook, call the creature's
*best cover* with the enemy's remembered position. That is not a search and does not move
anything: it tells the creature's cover bookkeeping which direction protection is wanted
from, so that the exposure values the rest of the combat layer reads are measured against
the right threat.

**Invariants** — it must be re-asserted every cycle because the enemy moves. Asserting it
once at entry produces a creature that keeps hiding from where the enemy used to be.

## `GetReadyToKillLowCover`

**Contract** — crouch, keep the eyes where they are, and bring the weapon up with the full
aiming animation. Raises the brain's cover-affecting flag for the duration and lowers it on
exit.

```text
FUNCTION initialize()
  base.initialize()
  brain.affect_cover := true

FUNCTION execute()
  base.execute()
  movement.body_state := crouch
  sight := keep the current direction
  aim_ready_force_full()

FUNCTION finalize()
  base.finalize()
  brain.affect_cover := false
```

**Invariants** — the posture is set per cycle, not at entry, because anything else in the
creature may stand it up between cycles and the whole value of this action is that the
creature is *down*.

**Notes** — the full aiming animation rather than the abbreviated one is the right choice
here for a visible reason: the creature is coming up from behind cover, and the short aim
would have the weapon appear already raised. See
[`stalker_combat_action_base.cpp`](stalker_combat_action_base.cpp.md).

The cover-affecting flag tells the squad's cover bookkeeping to count this creature as
occupying cover. All three actions raise it, which is the statement that a creature at a low
cover position is *using* that cover whatever posture it is in.

The eyes are left where they are rather than put on the enemy. A creature ducking to reload
or ready its weapon is not looking at anything.

## `KillEnemyLowCover`

**Contract** — stand up, track the enemy, fire, and keep the cover hint pointed at it.
Announces the engagement once on entry.

```text
FUNCTION initialize()
  base.initialize()
  movement.body_state := standing
  brain.affect_cover  := true
  play the attack line, allowed to start immediately, stopping within 4 to 6 seconds

FUNCTION execute()
  base.execute()
  sight := watch the enemy
  fire()
  IF no enemy selected THEN RETURN
  remembered := my memory of the enemy
  IF I do not remember it THEN RETURN
  best_cover_hint := remembered.position
```

**Invariants** — the posture is set once, at entry, unlike the crouch action. Standing is
the action's whole premise; a creature that got pushed down would be running a different
action.

**Notes** — firing goes through the aim gate in the combat action base, so the creature will
keep turning rather than shoot across the cover at nothing.

The attack line's timing window — may start at once, must stop between four and six
seconds — is the shape used for combat chatter throughout: a randomized stop so that a
squad's voices do not cut off together.

A build flag can compile the line out entirely, for testing combat without voices.

## `HoldPositionLowCover`

**Contract** — stand at the cover watching where the enemy was, for one to three seconds.
Fire only if the squad is flanking and there is something worth shooting at; otherwise just
hold the aim. When the interval elapses, **write three propositions directly into the combat
planner above** to hand control on.

```text
FUNCTION initialize()
  base.initialize()
  brain.affect_cover  := true
  movement.body_state := standing
  aim_ready()
  inertia_time := 1 second + uniform random up to 2 more

FUNCTION execute()
  base.execute()
  remembered := my memory of the selected enemy
  IF I do not remember it THEN RETURN
  sight := watch the remembered position

  IF the interval has elapsed THEN
    IF squad.may_detour()
       OR NOT squad.is_covering_a_detour()
       OR NOT fire_make_sense() THEN
      combat_planner.LookedOut      := true
      combat_planner.PositionHolded := true
      combat_planner.InCover        := false

  IF squad.is_covering_a_detour() AND fire_make_sense() THEN
    play the "need backup" line
    fire()
  ELSE
    aim_ready()

  refresh the cover hint from the enemy's remembered position
```

**Invariants** — the hand-off writes into the *combat planner's* storage, reached by casting
the brain's current action. That is a deliberate short circuit past the planner hierarchy,
and it is the only place in the stalker brain that does it. The three propositions belong to
the combat planner's own cover sequence, and this action is declaring, from two levels down,
that the sequence has advanced. A rebuild should make the channel explicit — a result the
sub-planner publishes upward, as every other sub-planner does — rather than reproducing the
reach-through.

**Notes** — the firing rule is the squad's, not the creature's. A stalker at low cover holds
its fire unless the squad is *covering a flank*, in which case it shoots to keep the enemy's
head down while somebody else moves, and calls for backup while doing it. That is
suppressive fire, produced by two squad queries and no coordination code.

The hand-off condition reads awkwardly and means something simple: advance the sequence
unless this creature is the one currently providing cover for a flank and actually has a
target. A creature doing that job stays where it is.
