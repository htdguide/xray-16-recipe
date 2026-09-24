# src/xrGame/stalker_property_evaluators.cpp

> The twenty questions a stalker's brain is allowed to ask about the world — one small object
> per question, each answering `true` or `false`.

**Needs** — [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`item_manager.h`](item_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [`ai/ai_monsters_misc.h`](ai/ai_monsters_misc.h.md) · [`alife_simulator.h`](alife_simulator.h.md) · [`alife_object_registry.h`](alife_object_registry.h.md) · [`alife_human_brain.h`](../xrServerEntities/alife_human_brain.h.md) · [`Actor.h`](Actor.h.md) · [`actor_memory.h`](actor_memory.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_enemy_manager.h`](agent_enemy_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`cover_point.h`](cover_point.h.md) · [`xrAICore/Navigation/level_graph.h`](../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`stalker_property_evaluators.h`](stalker_property_evaluators.h.md)
**Tier floor** — T2: predicate objects over already-gathered perception state; no device or format contact.

## Purpose

An evaluator is one half of the planner's vocabulary (chapter 14): it answers exactly one
yes/no question about the current world, and the planner's operators are written as
preconditions over those answers. This file holds every such question a **stalker** — the
human creature — can be asked. They are the measurement side of the behaviour model, and
they are load-bearing in the strongest sense: get one predicate wrong and the creature
still plans correctly, but plans about a world that is not there.

Three facts about them shape every entry below.

**They measure; they do not act.** The planner may call an evaluator many times inside one
plan search and may never call it at all, so an evaluator with a side effect makes the
creature's behaviour depend on the search's internal order. Two of the twenty break this
rule deliberately and are flagged where they occur.

**They are answered once per plan and cached.** The planner's contract (chapter 14) is that
a property is measured lazily on first use and then frozen for that plan, and that
re-planning is triggered by re-measuring exactly the properties the standing plan used. So
a question whose answer flickers frame to frame produces a creature that re-plans
constantly; several evaluators here are written with explicit hysteresis for that reason.

**Every one of them is bound to a creature and named.** The name is not cosmetic — it is
what the planner's failure dump prints, and a creature that has stopped deciding is
diagnosed by reading which named property came back with an unexpected value.

## State

Evaluators are stateless apart from the tuning each is constructed with.

```text
RECORD EnemyEvaluator            # the only one with memory of its own configuration
  grace_period  : int (ms)       # how long after losing an enemy the answer stays true
  suppress      : optional<bool> # an external flag that cancels the grace period

RECORD ReadyToKillEvaluator
  min_ammo      : int            # 0 means "do not consider ammunition at all"

CONSTANT wounded_enemy_reached_distance = 3.0   # world units
```

`wounded_enemy_reached_distance` is shared with the execution side of finishing off a
wounded enemy, so the "am I close enough" question and the action that walks there agree by
construction rather than by two copies of a number.

## The questions, grouped by who asks them

Each heading below is one evaluator. The heading states the question; the contract states
how it is answered and what makes the answer move.

### Existence and life

## `ALife`

**Contract** — *is the off-screen simulation running at all?* True when an alife simulator
exists. Asked once by the alife sub-planner, which is the creature's default branch: a
stalker in a level with no alife simulation (a multiplayer map, the editor) has nothing to
go to work at, and the branch must not be entered. Free of side effects and of cost.

## `Alive`

**Contract** — *am I alive?* Reads the creature's own liveness flag. This is the property
the root planner's death branch turns on, so it is the single most consequential predicate
in the file: the entire behaviour tree hangs off its negation.

### Enemies

## `Enemies`

**Contract** — *is there an enemy, or was there one recently?* True immediately if the
memory layer currently has a selected enemy. Otherwise true while the configured grace
period has not elapsed since the last enemy was held — unless the suppression flag the
evaluator was handed is raised, in which case the grace period is skipped and the answer is
false at once.

```text
FUNCTION evaluate() -> bool
  IF memory.enemy.selected EXISTS
    RETURN true
  IF suppress EXISTS AND suppress IS true
    RETURN false                                  # an external decision to drop combat now
  RETURN now < memory.enemy.last_enemy_time + grace_period
```

**Notes** — the grace period is what stops a stalker falling out of combat the instant it
loses sight of its target, and it is supplied by the caller rather than fixed here, so the
same class serves three different registrations: the root planner asks with the combat
planner's own post-combat wait, the combat planner registers it twice — once with a grace
period of zero, under the name of the *pure* enemy property, and once with the delay, under
the plain enemy property — and the kill-wounded planner asks with the default. The pair
"pure enemy" versus "enemy" is exactly the distinction between *there is one now* and *I am
still in a fight*, and operators throughout combat are conditioned on one or the other
according to which they mean.

## `SeeEnemy`

**Contract** — *can I see my enemy right now?* False when no enemy is selected; otherwise
the vision layer's current-visibility answer for that enemy. Note "right now": this is live
perception, not memory, and it flips the moment the enemy steps behind cover.

## `EnemySeeMe`

**Contract** — *can my enemy see me right now?* Asks the *enemy's* own vision memory about
*this* creature. Answers false when there is no enemy, or when the enemy is neither a
creature nor the player — that is, when it has no perception of its own to consult.

**Notes** — this is the one place a stalker reaches into another entity's perception rather
than its own, and it is what makes flanking legible: the creature can know it is unobserved
without the engine simulating a separate "am I hidden" sense. A rebuild that gives every
entity a uniform senses interface removes the two-case dispatch here entirely.

## `EnemyCriticallyWounded`

**Contract** — *is my enemy a stalker who has been knocked into the critically-wounded
state?* False for no enemy and for any enemy that is not a stalker — animals and the player
have no such state. Feeds the branch that walks up to a downed enemy and finishes it,
rather than continuing to shoot from cover.

## `EnemyReached`

**Contract** — *am I the one who should finish this wounded enemy off, and am I close
enough to do it?* Two conditions, in this order: the squad's agent layer must name **this**
creature as the processor for that wounded enemy, and the straight-line distance must be
within `wounded_enemy_reached_distance`.

```text
FUNCTION evaluate() -> bool
  enemy := memory.enemy.selected
  IF enemy IS none
    RETURN false
  IF agent_manager.enemy.wounded_processor(enemy) IS NOT my_id
    RETURN false                                   # somebody else in the squad owns this kill
  RETURN distance(my_position, enemy.position) <= wounded_enemy_reached_distance
```

**Notes** — the ownership check is the whole point. Without it every member of a squad
converges on the same downed body. The squad layer arbitrates once and every member's
planner then reads the same answer, which is how a group behaviour is produced from
per-creature planning with no group plan anywhere.

## `TooFarToKillEnemy`

**Contract** — *is my enemy beyond the effective range of my best weapon?* False when there
is no enemy or no weapon at all. Otherwise it is asked of the creature's remembered
position for that enemy, not the enemy's true position — a creature closes the distance to
where it last believed the enemy was, which is what makes it possible to walk into an
ambush.

## `PlayerOnThePath`

**Contract** — *is the player standing in my way?* All four must hold: some enemy is
selected (so the creature is in combat at all), the player is an enemy of this creature, the
player is visible right now, and the player intersects the creature's intended path within
two world units. Drives the "push past" behaviour.

**Notes** — the first condition is about combat, not about the player. A friendly stalker
walking a patrol does not shove the player aside; only a hostile one already fighting does.

### Weapons and ammunition

## `ItemToKill`

**Contract** — *do I have anything that could kill with?* Delegates to the creature's own
weapon bookkeeping.

## `ItemCanKill`

**Contract** — *is the thing I am holding actually capable of killing right now?* Distinct
from having one: a weapon with no ammunition passes `ItemToKill` and fails this.

## `FoundItemToKill`

**Contract** — *do I remember a weapon lying somewhere I could go and take?* The
precondition of the branch that sends an unarmed stalker to fetch a gun.

## `FoundAmmo`

**Contract** — *do I remember ammunition I could go and take?* Same shape, for the branch
that sends a stalker with an empty weapon to fetch rounds.

## `ReadyToKill`

**Contract** — *can I open fire this instant?* The creature must report itself ready and
must have a best weapon. If the evaluator was constructed with no ammunition floor, that is
the whole answer. Otherwise a magazine at or below the floor answers false — with one
exception: when the *magazine capacity itself* is at or below the floor, the floor is
meaningless for that weapon and the creature is ready as long as it is not mid-reload.
Being mid-reload always answers false.

```text
FUNCTION evaluate() -> bool
  IF NOT ready_to_kill OR best_weapon IS none
    RETURN false
  IF min_ammo == 0
    RETURN true
  IF best_weapon.rounds_loaded <= min_ammo
    IF best_weapon.magazine_size <= min_ammo
      RETURN best_weapon.state IS NOT reloading   # the floor cannot be met by this weapon at all
    RETURN false
  RETURN best_weapon.state IS NOT reloading
```

**Notes** — the magazine-capacity escape is the load-bearing line. Without it a creature
carrying a weapon whose magazine is smaller than the floor would report "not ready"
forever and stand still in a firefight. The floor exists so that a creature holding a
nearly-empty weapon prefers to reload or to reposition before committing to an exchange;
it is a per-registration number, and the only non-zero one in the shipped set is six, used
inside smart cover.

## `ReadyToKillSmartCover`

**Contract** — the same question, asked of a creature occupying a smart cover. It answers
true unconditionally when the currently occupied cover cannot be fired from, and otherwise
falls through to `ReadyToKill`.

**Notes** — the unconditional true reads backwards and is deliberate. Inside a smart cover
the property does not mean "I can shoot"; it means "nothing about my weapon is what is
stopping me", so that the cover's own animation planner, not the weapon state, decides when
the creature leans out. A cover with no firing loophole must not leave the creature stuck
waiting to become ready.

### Hazards

## `Anomaly`

**Contract** — *is there an anomaly near me I have not yet accounted for?* False when the
creature knows of no undetected anomaly. With no enemy selected, an undetected anomaly is
enough. **With** an enemy selected, the answer is gated on the group's combat assessment:
the anomaly wins only if the group judges the fight unwinnable. A stalker under fire walks
around a gravity anomaly only when it was going to break off anyway.

## `InsideAnomaly`

**Contract** — *am I standing in an anomaly right now?* Same two-case shape as `Anomaly`,
with the same combat gate: being inside an anomaly does not by itself interrupt a fight the
group believes it is winning.

## `Panic`

**Contract** — *should I break off and run?* False outright while an animation with a global
selector is playing — a creature in the middle of a scripted or full-body animation cannot
flee. Otherwise it is the group combat assessment, read the other way round: panic is
exactly the case where the assessment does *not* return "engage".

**The group combat assessment.** The same query backs all three hazard evaluators, and it is
worth stating once. It asks the squad-group registry: *can the members of my group within
300 world units defeat the enemies we can currently see, with at least my panic threshold
of confidence?* The probability comes from a data-authored expression evaluated pairwise
over members and enemies. The answer is cached per group for two seconds and reused by every
member that asks inside that window, which is what makes a squad break and run *together*
rather than one creature at a time. A creature's panic threshold is its own tuned number, so
a veteran and a novice standing side by side can read the same group probability and reach
different answers.

**Notes** — the caching window is genuinely shared mutable state reached through a global
registry, and it is a side effect: the first member of a group to ask each cycle pays for
the query and pins the answer for the rest. That the result is a *group* decision cached
for two seconds — not a per-creature one recomputed per plan — is the single most visible
thing about squad behaviour in this game, and a rebuild that makes it per-creature produces
squads that dissolve one member at a time.

### Errands

## `Items`

**Contract** — *is there an item I have decided to go and pick up?* True when the memory
layer's item selector has settled on one. Note that the choosing already happened
elsewhere; the evaluator only reports that a choice exists.

## `SmartTerrainTask`

**Contract** — *has a smart terrain given me a job?* Requires the alife simulation to be
running, and the creature to have a server-side human record. **Asks that record's brain to
select a task**, then reports whether a smart terrain now owns it.

**Notes** — this is the second evaluator with a side effect, and the larger of the two: a
measurement triggers the off-screen brain's task selection, so merely *asking* whether the
creature has a job is what gives it one. It is also the seam where the client-side planner
reaches across into the server-side alife record — the stalker standing in the level and the
record the alife simulation advances are the same entity, and this is the line where the
live creature asks the authoritative record what it is supposed to be doing. A rebuild that
separates measurement from task assignment must run the selection somewhere else in the
cycle, and must run it *before* the plan search, or the creature will be a full planning
cycle behind its orders.

## `ReadyToDetour`

**Contract** — *am I in a state where circling around the enemy makes sense?* Delegated
wholesale to the creature.

## `ShouldThrowGrenade`

**Contract** — *should I throw a grenade at my enemy now?* A gate chain, each link a reason
not to, ending in a side effect: the target is recorded on the creature before the answer
comes back. It is latching — once the throw has started, the answer stays true regardless of
everything below it, so that a throw in progress is never abandoned mid-animation.

```text
FUNCTION evaluate() -> bool
  IF world_state.started_to_throw_grenade
    RETURN true                                    # latched: never interrupt a throw

  # only from a settled position, never while moving up
  IF NOT (in_cover OR position_holded OR enemy_detoured)
    RETURN false

  IF last_throw_time + throw_interval >= now       # rate limit, per creature
    RETURN false
  IF grenade_slot IS empty
    RETURN false

  enemy := memory.enemy.selected
  IF enemy IS none OR NOT enemy.is_human
    RETURN false
  IF visible_now(enemy)
    RETURN false                                   # if I can see him I can shoot him

  remembered := memory.memory(enemy)
  IF remembered IS none
    RETURN false
  IF distance(my_position, remembered.position) < 10
    RETURN false                                   # I would be inside the blast

  IF NOT agent_manager.member.can_throw_grenade(remembered.position)
    RETURN false                                   # a squadmate is in the way

  record_throw_target(remembered.position, remembered.vertex, enemy)   # side effect

  IF grenade_slot_item IS the_active_item
    RETURN true                                    # already committed; cannot back out
  RETURN throw_trajectory_is_clear
```

**Notes** — four of these gates are the grenade's entire character. *Only at rest*: a
grenade is thrown from cover or from a held position, never on the move. *Only when the
enemy is unseen*: grenades are for flushing someone out, not for a target already in the
sights. *Only beyond ten units*: the creature will not blow itself up. *Only with squad
clearance*: the agent layer refuses a throw that would land on a squadmate, which is what
keeps a squad from killing itself in a corridor.

Recording the target inside the evaluator is the side effect that matters most in this file:
the throw action later reads a target that the *question* set. A rebuild that evaluates
this predicate speculatively during the plan search, or evaluates it and then discards the
plan, leaves a stale throw target behind. The cleanest correction is to have the evaluator
answer from the same computation but publish the target only when the action begins.

The second commitment check — active item already being the grenade — exists because
swapping back from a drawn grenade is not animatable; past that point the creature must
throw whether or not the trajectory is clear.

## `LowCover`

**Contract** — *am I using cover low enough that I must rise to fire?* **Always answers
false.** The real computation — compare the cover value of the occupied navigation vertex
in the enemy's direction at standing height against the value at crouch height, and report
low cover only when crouching is genuinely better — is present but disabled, and its author
left a note that further conditions were still needed.

**Notes** — the property is not vestigial: a whole sub-planner is registered against it and
is therefore unreachable in the shipped game. A rebuild must decide deliberately whether to
ship the disabled branch. Reviving it changes visible behaviour — creatures would start
popping up from behind low walls to shoot — so the honest reading is that this was cut
during tuning rather than forgotten.

## What could not be recovered

- The ten-unit grenade self-blast radius, the two-unit "player is in my way" width and the
  three-unit wounded-enemy reach distance are all bare numbers with no derivation in the
  source; only the last is shared with anything else.
- The two-second cache window and the 300-unit group radius of the combat assessment appear
  identically at all three call sites, written out long-hand rather than named, so it is not
  recoverable whether they were meant to be one tunable or three coincidences.
- Why the `LowCover` computation was disabled, and what the "several other conditions" its
  author noted were.
