# src/xrGame/stalker_kill_wounded_actions.cpp

> Executing a downed enemy: who is allowed to do it, which weapon does it, what is said first, and the guarantee that it actually happens.

**Needs** — [`stalker_kill_wounded_actions.h`](stalker_kill_wounded_actions.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_enemy_manager.h`](agent_enemy_manager.h.md) · [`Inventory.h`](Inventory.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`sound_player.h`](sound_player.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`stalker_kill_wounded_actions.h`](stalker_kill_wounded_actions.h.md)
**Tier floor** — T1: the final step constructs a damage message in the wire format and dispatches it to itself.

## Purpose

A downed but not dead enemy is a distinct game state, and killing one is a small scripted
scene rather than a shot. The five actions are that scene. Three things in this file are
load-bearing beyond the sequence itself: the squad-wide claim that stops six stalkers
executing the same body, the choice of *which* weapon does it, and the fallback that makes
the kill happen even when the geometry will not let a bullet reach.

## State

`Stateless.` Everything that has to be remembered — who is executing whom, and whether it
is already done — lives in the squad coordinator, because it has to be shared.

## `weapon_to_kill` (shared helper)

**Contract** — which weapon a creature executes with. Pure.

```text
FUNCTION weapon_to_kill(creature) -> Item
  pistol := creature.inventory.item_in_sidearm_slot
  IF there is no sidearm            THEN RETURN creature.best_weapon
  IF the sidearm is not a magazine weapon THEN RETURN creature.best_weapon
  IF the sidearm cannot kill (empty, broken) THEN RETURN creature.best_weapon
  RETURN the sidearm
```

**Notes** — a stalker executes with its **pistol** when it has a usable one, and with its
primary only otherwise. This is pure characterization and it is worth keeping: the
sidearm is what a soldier draws for a body on the ground, and switching weapons is what
gives the scene its beat.

## `should_process` (shared helper)

**Contract** — may this creature execute this enemy. Reads the squad coordinator; pure with
respect to the creature.

```text
FUNCTION should_process(creature, enemy) -> bool
  IF the squad records this enemy as already processed THEN RETURN false
  claimant := the squad's recorded executioner for this enemy
  IF a claimant exists AND it is not me THEN RETURN false
  RETURN true
```

**Invariants** — the claim is squad-wide, single-holder, and checked at *every* step of the
sequence rather than once at the start. That is what makes the behaviour robust: a creature
that loses the claim mid-approach stops advancing rather than arriving and executing a
second time.

**Notes** — this is the whole mechanism preventing a squad from converging on one body. It
is also why the approach action has three different movement outcomes rather than one.

## `ReachWounded` — entry

**Contract** — clear the two later progress propositions, configure a relaxed walk toward
the enemy, and hold the execution weapon idle with a burst shape of at most one round every
one to one and a half seconds.

**Invariants** — the mental state is *at ease*, not danger. A stalker walking up to a downed
enemy walks; anything else reads as a threat response to something already beaten.

**Notes** — the burst shape — minimum zero, maximum one round, a full second between — is
fixed in code rather than drawn from the creature's tuned table, and it is used by all five
actions. It is what makes an execution a deliberate single shot instead of a burst.

## `ReachWounded` — per cycle

**Contract** — steer to the enemy's remembered navigation vertex, and decide whether to keep
walking or stop, on four different grounds.

```text
FUNCTION execute()
  base.execute()
  IF no enemy selected THEN RETURN
  enemy := selected enemy

  IF the squad says this enemy is already processed THEN stand still; RETURN
  remembered := my memory of the enemy
  IF I do not remember it THEN stand still; RETURN

  IF the enemy's navigation vertex is somewhere I may go THEN
    steer to that vertex
  ELSE
    head for the nearest position to it that I may occupy

  IF should_process(me, enemy) THEN walk; RETURN     # it is mine: close in
  claimant := the squad's executioner for this enemy
  IF there is no claimant THEN stand still; RETURN   # nobody's yet: wait
  IF I am within 3 units of the enemy THEN stand still; RETURN   # someone else's: keep back
  walk                                               # someone else's, but far: come closer
```

**Invariants** — the destination is always set, even in the branches that then stand still.
That keeps the creature facing the right way and ready to resume the moment the claim
resolves.

**Notes** — the three-unit standoff is the spectator rule: squad members that are not the
executioner walk up to a respectful distance and stop, which is what produces the small
crowd around a downed enemy without any group behaviour being written. A commented-out
alternative measured the distance to the *executioner* rather than to the body; the shipped
rule uses the body, which behaves better when the executioner is still approaching from the
far side.

## `AimWounded`

**Contract** — stand still, bring the execution weapon to aimed-and-ready, put the eyes on
the enemy, and **take the squad claim**. Reports the aim complete only when four things all
hold.

```text
FUNCTION initialize()
  base.initialize()
  movement.mental_state := danger
  movement.body_state   := standing
  movement.gait         := stand still
  weapon_goal(AIM_READY, weapon_to_kill, the execution burst shape)
  sight := watch the enemy
  squad.mark_processed(enemy, true)                  # claim it
  IF the enemy is not visible right now THEN movement.gait := walk

FUNCTION execute()
  base.execute()
  IF this is the first cycle          THEN RETURN
  IF the settle timer has not elapsed THEN RETURN
  IF the claim is no longer mine      THEN RETURN
  IF the head has not reached its target yaw and pitch THEN RETURN
  IF the enemy is not visible right now THEN RETURN
  set property WoundedEnemyAimed = true
```

**Invariants** — the aim is judged by the *head* actually having arrived at its target
angles, not by a timer alone. Both the timer and the head test must pass, which is what
stops the creature firing during the turn.

Skipping the first cycle is deliberate: on the entry cycle the head has not been given its
target yet, so its current and target angles are trivially equal and the test would pass
spuriously.

**Notes** — if the enemy is not visible from where the creature stopped, the gait is set
back to walking. That is the recovery for an enemy that went down behind something: the
creature keeps closing until it can see the body. The action has no destination of its own,
so it walks along whatever the approach action left set.

The settle interval is one second, configured by the planner rather than by the action.

## `PrepareWounded`

**Contract** — stand still, aim, and **say the line**. Reports prepared only once the voice
line has finished playing.

```text
FUNCTION initialize()
  base.initialize()
  stand still, standing, mental state danger
  play the execution voice line
  weapon_goal(AIM_READY, weapon_to_kill, the execution burst shape)

FUNCTION execute()
  base.execute()
  IF no enemy selected THEN RETURN
  IF the claim is not available to me THEN
    mask out the execution voice channel        # someone else is speaking; stay quiet
    RETURN
  IF the squad's executioner is not me THEN RETURN
  sight := watch the enemy
  IF no voice line is still playing THEN set property WoundedEnemyPrepared = true

FUNCTION finalize()
  base.finalize()
  unmask the execution voice channel
```

**Invariants** — the step completes on the *sound* finishing, not on a timer. The line's
length is authored per creature and per faction, so a timer would either cut it off or leave
a silence. This is one of the few places in the AI where an audio asset's duration is a
control-flow input, and a rebuild must keep the dependency rather than approximate it.

**Notes** — masking the voice channel for non-executioners is why only one stalker in a
group delivers the line. The mask is cleared on exit whatever happened, so a creature that
loses the claim is not left mute.

This action is the one that makes the whole branch worth its complexity. Without it the
sequence is "walk over and shoot"; with it, a stalker stands over a wounded man, says
something, and then shoots.

## `KillWounded`

**Contract** — fire the execution shot, and if the shot cannot physically land, apply the
damage directly. Sets the post-kill pause flag on entry so the pause is guaranteed to
follow.

```text
FUNCTION initialize()
  base.initialize()
  set property PausedAfterKill = true        # arm the pause before the kill
  stand still, standing, mental state danger

FUNCTION execute()
  base.execute()
  IF no enemy selected THEN RETURN
  enemy := selected enemy
  sight := watch the enemy
  weapon_goal(FIRE, weapon_to_kill, the execution burst shape)

  IF nothing is in my hands THEN RETURN
  IF the enemy is visible now
     AND I can hit it
     AND I would not hit an ally THEN RETURN        # the real shot will do it

  # otherwise: deliver the damage without a projectile
  build a damage message:
    target      := enemy
    source      := me
    weapon      := weapon_to_kill
    direction   := straight ahead
    bone        := the enemy's root bone
    power       := 1, impulse := 1
    damage kind := the kind that finishes a downed creature
  dispatch it as if it had arrived over the network
```

**Invariants** — arming the pause in *entry* rather than at the kill means the pause happens
even if the action is abandoned after firing. The sequence cannot end with a stalker
snapping instantly back to patrolling over a fresh body.

**Notes** — the direct-damage fallback is the file's one genuinely unpleasant piece of
engineering, and the original says so in its own comment. A downed creature may be playing
an animation that puts its body inside another object — a doorway, a crate, another
corpse — so that no ray from the executioner reaches a hittable bone. Rather than leave the
sequence stuck forever, the creature fabricates the damage event and sends it through the
normal damage path.

Two things about the fabrication matter to a rebuild. It goes through the **same message
path a networked hit would take**, so every listener — scoring, reputation, alife
bookkeeping, the corpse's own death handling — sees an ordinary kill; nothing special-cases
it. And the damage kind is the one reserved for finishing a downed creature, so armour and
resistances do not apply and the kill is certain. A rebuild that applies damage by calling a
method directly will silently skip whatever the message path does on the way.

## `PauseAfterKill`

**Contract** — stand over the body for a second with the weapon up and the eyes wherever
they already point, then clear the pause flag and let the branch end.

**Notes** — the sight mode here is "keep the current direction", the only action in the
sequence that does not track the enemy. The creature has finished with it; holding the aim
on a corpse would read as menace rather than as a beat. The interval is one second,
configured by the planner.
