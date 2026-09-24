# src/xrGame/stalker_combat_actions.cpp

> Every leaf behaviour of a firefight: arming, taking cover, shooting, looking out, holding, flanking, fleeing, grenades and the moments either side of the fight.

**Needs** — [`stalker_combat_actions.h`](stalker_combat_actions.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`stalker_planner.h`](stalker_planner.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md) · [`visual_memory_manager.h`](visual_memory_manager.h.md) · [`hit_memory_manager.h`](hit_memory_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [`cover_manager.h`](cover_manager.h.md) · [`cover_evaluators.h`](cover_evaluators.h.md) · [`cover_point.h`](cover_point.h.md) · [`stalker_movement_restriction.h`](stalker_movement_restriction.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`agent_location_manager.h`](agent_location_manager.h.md) · [`danger_cover_location.h`](danger_cover_location.h.md) · [`detail_path_manager.h`](detail_path_manager.h.md) · [`level_path_manager.h`](level_path_manager.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md) · [`WeaponMagazined.h`](WeaponMagazined.h.md) · [`Missile.h`](Missile.h.md) · [`sound_player.h`](sound_player.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../xrAICore/Navigation/ai_object_location.h.md) · [Seam: Static collision database](../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`stalker_combat_actions.h`](stalker_combat_actions.h.md)
**Tier floor** — T2: several cover-database searches and a collision ray per creature per cycle.

## Purpose

The eighteen leaf behaviours of the combat branch. The planner in
[`stalker_combat_planner.cpp`](stalker_combat_planner.cpp.md) decides which one runs; this
file is what each one actually does to the creature.

Six of them — get ready, take cover, kill, look out, hold, flank — form one loop, and they
share a great deal. Read those six together: their differences are small and every
difference is a decision.

## Shared machinery

Three things recur and are stated once here so the sections can be short.

**The cover hint.** Almost every action calls the creature's *best cover* with the enemy's
remembered position on every cycle. That is not a search: it declares which direction
protection is wanted from, and the creature's cover bookkeeping — shared with the squad —
answers with a point. It must be re-asserted per cycle because the enemy moves.

**The look-down guard.** Several actions, when aiming at a *remembered* rather than visible
enemy, check two things: has it been more than three seconds since the enemy was last seen,
and is the remembered position more than three and a half units above or below the
creature. If both, the aim is redirected to a point at the remembered horizontal position
but at the creature's own eye height. Without it a stalker tracking an enemy that went up a
staircase ends up staring at the ceiling, which looks broken and is. A rebuild needs some
form of this: the memory of a position is not a memory of a sight line.

**The exposure probe.** A short helper casts a ray ten units forward from the creature's
eyes against the static collision database and returns the distance to what it hits, or a
large value if it hits nothing. Three or more units of clear space means "I can see out";
less means "something is in my face". Actions use it to decide whether the creature has
successfully looked out of its cover. This is a *geometric* test standing in for a tactical
one, and it is remarkably cheap for what it buys — no visibility graph, no cover value, just
"is there a wall in front of my eyes".

## `GetItemToKill`

**Contract** — walk to the best weapon the creature has spotted lying around, watching it
the whole way. Does nothing if no such item is known.

```text
FUNCTION initialize()
  base.initialize()
  silence all but the humming channel
  sight := watch the spotted weapon
  movement.mental_state := danger

FUNCTION execute()
  base.execute()
  IF no spotted weapon THEN RETURN
  steer to the weapon's navigation vertex and position
  movement.path_type := level path, smooth detail
  movement.body_state := keep crouching if crouched, else stand
  movement.gait := walk
  weapon_goal(IDLE)

FUNCTION finalize()
  base.finalize()
  sight := clear, then watch along the path
  IF alive THEN unmask sounds
```

**Notes** — posture is *preserved* rather than set: a creature crouched behind cover crawls
to the weapon crouched. Everywhere else in the file posture is a decision; here it is
inherited, because picking up a weapon is not itself a tactical act.

## `MakeItemKilling`

**Contract** — walk to ammunition for the weapon the creature already holds, while keeping
one eye on the path and one on the enemy.

```text
FUNCTION initialize()
  base.initialize()
  silence all but the humming channel
  sight := clear, then install TWO sight actions:
      watch-item  : look along the path,   weight 1, over 3 seconds
      watch-enemy : look at the enemy's position, weight 1, over 3 seconds
  movement.mental_state := danger

FUNCTION execute()
  base.execute()
  IF no spotted ammunition THEN RETURN
  steer to the ammunition
  movement.body_state := standing;  movement.gait := walk
  update the watch-enemy sight action with the enemy's current position
  weapon_goal(IDLE)
```

**Invariants** — this is the only action in the file that installs *two* competing sight
actions rather than one. The sight system blends them by weight over the stated interval, so
the creature's head alternates between where it is going and where the enemy is. Anything
else — picking one — either walks the creature into a wall or loses the enemy.

## `RetreatFromEnemy`

**Contract** — run away in a panic, toward *far* cover, firing over the shoulder if the
enemy is visible and there is nowhere to run to. Announces itself with the panic line.

```text
FUNCTION execute()
  base.execute()
  IF no enemy selected THEN RETURN
  movement.gait         := run
  movement.path_type    := level path, smooth detail
  movement.mental_state := panic
  movement.body_state   := standing

  remembered := my memory of the enemy
  IF I remember it THEN
    configure the far-cover evaluator: away from remembered.position, out to 300 units
    point := best_cover(near = my position, radius = 30, far evaluator)
    IF point is none THEN retry with radius 50

  IF point exists THEN
    steer to point;  weapon_goal(IDLE);  sight := watch along the path
  ELSE IF the enemy is visible now THEN
    movement.mental_state := danger      # nowhere to run: stop panicking and fight
    fire();  sight := watch the enemy
  ELSE
    weapon_goal(IDLE);  sight := watch the cover

  play the panic line
```

**Invariants** — the search radii here are three and five times the ones every other action
uses, and the evaluator's own range is three hundred units. Retreat is the only behaviour
that wants cover *far away*; everything else wants cover nearby.

**Notes** — the fall-back from panic to alarm when there is nowhere to go is the file's
best small decision. A cornered stalker stops fleeing and fights, without any "cornered"
state existing.

## `weight` (of `RetreatFromEnemy`)

**Contract** — the cost this operator contributes to the planner's search. Fixed at one
hundred, against a default of one.

**Invariants** — this is the only operator in the stalker brain that overrides its weight.
Planning is a shortest-path search over operators, so a weight of one hundred makes retreat
the *last* plan the search will accept: it is chosen only when no other route to the goal
exists at all. Priority through cost rather than through preconditions, used exactly once.

## `GetReadyToKill`

**Contract** — move onto the cover point the creature's bookkeeping has chosen and bring the
weapon up. Constructed in two variants by a flag: the *primary* variant also resets the four
cover-sequence propositions and freezes enemy selection; the *detour* variant does neither
and uses the full aiming animation.

```text
FUNCTION initialize()
  base.initialize()
  remember my current posture
  movement.path_type := level path, smooth detail
  movement.head_for_nearest_accessible_position()
  movement.mental_state := danger
  movement.body_state   := the remembered posture
  IF this is the primary variant THEN
    clear InCover, LookedOut, PositionHolded, EnemyDetoured
    remember whether enemy re-selection was enabled, then disable it

FUNCTION execute()
  base.execute()
  IF I have no weapon, or no enemy THEN RETURN
  IF distance still to walk < 2 THEN
    movement.gait := walk;  sight := keep the current direction
  ELSE
    movement.gait := run;   sight := watch along the path
  movement.body_state := standing            # see Notes

  remembered := my memory of the enemy
  IF I remember it THEN
    point := best_cover_hint(remembered.position)
    IF point exists THEN
      setup_cover(point)
      brain.affect_cover := (I have arrived, or am within 1 unit of it)
    ELSE
      brain.affect_cover := true
      movement.gait := stand still
      movement.head_for_nearest_accessible_position()

  IF primary variant THEN aim_ready() ELSE aim_ready_force_full()
  IF path completed THEN allow the cover bookkeeping to try advancing

FUNCTION finalize()
  base.finalize()
  IF primary variant THEN restore the enemy-re-selection setting
```

**Invariants** — **freezing enemy selection** for the duration is the important half of the
primary variant. Readying a weapon takes a second or two; if the creature re-picked its
enemy mid-animation it would restart, and a squad under fire would never get a shot off. The
previous setting is saved and restored rather than forced back on, because something else
may legitimately have it disabled.

**Notes** — the posture guard reads as `if the remaining distance is greater than minus ten
units, stand up`, which is always true: the constant is negative and a distance is not. So
the creature always stands. The negative constant is a switched-off feature — crouching for
the last stretch of the approach — left in a form that compiles. Two more traces of it sit
next to it as commented-out posture and gait lines. A rebuild should stand, and drop the
constant.

`affect_cover` tells the squad's cover accounting whether this creature is *actually* at its
point; it is set true on arrival and false while still moving, so a creature running to cover
does not have that cover counted as held.

"Try advancing" is the cover bookkeeping's bounding mechanism: a creature that has arrived
signals that it could move up to a more forward point if one exists.

## `KillEnemy`

**Contract** — stand and shoot. Resets three of the four sequence propositions on entry,
announces the engagement, and claims the cover.

```text
FUNCTION initialize()
  base.initialize()
  movement.path_type := level path, smooth detail
  movement.head_for_nearest_accessible_position()
  movement.mental_state := danger
  movement.gait := stand still
  clear LookedOut, PositionHolded, EnemyDetoured
  play the attack line
  brain.affect_cover := true

FUNCTION execute()
  base.execute()
  enemy := selected enemy
  IF there is no live enemy THEN
    sight := watch along the path
    RETURN
  refresh the cover hint from the enemy's remembered position
  IF the enemy is visible now THEN
    sight := watch the enemy;  fire()
  ELSE
    aim_ready()
    sight := watch the enemy's remembered position
```

**Invariants** — the creature does **not** fire at a remembered position. It aims there and
waits. That is a deliberate correction to earlier behaviour in which a stalker would empty
magazines into a wall it believed an enemy was behind.

**Notes** — the planner instantiates this same class three times under three operator names
— the ordinary kill, "kill if the enemy cannot see me", and "kill if the enemy is critically
wounded". The behaviour is identical; only the preconditions differ. That is the right shape:
the *when* is the planner's business and the *what* is the action's.

## `TakeCover`

**Contract** — walk to the cover point, shooting on the way when the enemy is visible,
and report `InCover` on arrival. Announces itself only when close to a visible human enemy
and in a group.

Structurally the same as `GetReadyToKill`'s execution, with three differences: the gait is a
walk rather than distance-dependent; arrival sets `InCover` as well as allowing the advance;
and the not-visible branch aims at the remembered position through the look-down guard rather
than merely aiming ready.

**Notes** — the entry voice line is gated on four conditions at once: the enemy is human, the
squad permits this member to speak, the enemy is within ten units, and the creature is in a
group whose behaviour is coordinated. It is the "backup" call — a stalker shouting for help
as it breaks contact at close range — and the four gates are what stop it being shouted
constantly.

## `LookOut`

**Contract** — shift from cover to somewhere with a line of sight on the enemy, crouched or
standing. Reports `LookedOut` as soon as the exposure probe says the creature can see out.

```text
FUNCTION initialize()
  base.initialize()
  IF at least 5 seconds since the last posture choice THEN
    set property UseCrouchToLookOut := private_random_boolean()
    remember the time
  movement.path_type := level path, smooth detail
  movement.mental_state := danger
  movement.body_state := crouch or stand, per that property
  movement.gait := walk
  movement.head_for_nearest_accessible_position()
  IF the creature is ready to flank THEN
    aim_ready()
  ELSE
    aim_ready_force_full();  movement.gait := stand still
  inertia_time := 1 second
  brain.affect_cover := true

FUNCTION execute()
  base.execute()
  IF no enemy, or I do not remember it THEN RETURN
  sight := the enemy's remembered position, through the look-down guard
  refresh the cover hint

  IF exposure_probe() >= 3 THEN
    stop where I am
    set property LookedOut = true
    RETURN

  configure the close-cover evaluator against the remembered position
  point := best_cover(near = my position, radius = 10, close evaluator)
  IF point is none OR (point is where I stand AND path completed) THEN
    retry with radius 30
  IF point exists THEN steer to it ELSE head for nearest accessible position
```

**Invariants** — the five-second rate limit on the crouch choice is what stops the creature
switching posture every time the planner re-enters this action, which under fire can be
several times a second. The choice is *sticky in time*, not per action instance.

**Notes** — the private, cycle-counter-seeded random source exists so that squad members
choose independently; see
[`stalker_danger_in_direction_actions.cpp`](stalker_danger_in_direction_actions.cpp.md) for
the same device.

If the creature is not yet ready to flank, it uses the full aim animation and stops moving.
That couples two things that look unrelated: a creature still bringing its weapon into the
flanking carry should not be shuffling toward a firing position at the same time.

This version's cover search deliberately passes **no movement restrictor**, unlike its twin
in the danger branch. A creature looking out of combat cover may step somewhere it would not
normally path to. There is no comment explaining it and it may be an oversight; it is a real
behavioural difference and is recorded here as one.

The arrival test that would have set `LookedOut` on reaching the chosen point is commented
out, leaving the exposure probe as the only way this action completes. The creature therefore
keeps shifting until it actually has a clear line, rather than until it reaches a nominally
good spot — which is the better rule, and is presumably why the other was removed.

## `HoldPosition`

**Contract** — wait at the cover for one to three seconds, aiming at the enemy's last known
position, then hand the sequence on if the squad allows. Fires only while covering somebody
else's flank.

Its body is the same as the low-cover hold described in
[`stalker_low_cover_actions.cpp`](stalker_low_cover_actions.cpp.md), with two differences: it
writes into its *own* planner's storage rather than reaching up into another planner's, and
it retracts `LookedOut` when the exposure probe says the creature has lost its line of sight.

**Invariants** — the hand-off writes `PositionHolded = true` and `InCover = false` together.
Clearing `InCover` is what makes the flank reachable, since the flanking action requires
*not* being in cover.

## `DetourEnemy`

**Contract** — flank, in bounds, at a run. On entry: tell the squad, drop the cover claim,
**publish the abandoned cover as a danger location**, and call the flank out loud.

```text
FUNCTION initialize()
  base.initialize()
  tell the squad I am flanking
  movement.path_type := level path, smooth detail
  movement.mental_state := danger
  movement.body_state := standing;  movement.gait := run
  aim_ready()
  IF I hold a cover point THEN
    squad.locations.add(DangerLocation(that point, now, 120 s, 5 units,
                                       visible only to my own squad))
  release my cover claim
  IF the enemy is human and my group coordinates THEN play the flanking line

FUNCTION execute()
  base.execute()
  IF no enemy, or I do not remember it THEN RETURN
  IF path completed THEN
    configure the angle evaluator: the remembered position, standoff 10,
        my weapon's range, the enemy's navigation vertex
    point := best_cover(near = my position, radius = 10, angle evaluator)
    IF point is none THEN retry with radius 30
    IF point exists THEN steer to it ELSE head for nearest accessible position
    IF path completed THEN set property EnemyDetoured = true
  sight := the enemy's remembered position, through the look-down guard

FUNCTION finalize()
  base.finalize()
  IF alive THEN tell the squad I am no longer flanking
```

**Invariants** — publishing the abandoned cover as a danger location, visible **only to the
creature's own squad**, is the mechanism that stops a flanking manoeuvre collapsing: without
it an ally immediately takes the point the flanker just left, and the flanker's own pathing
may route it back through there. The publication is masked to the squad because the position
is not dangerous to anybody else — it is *reserved against re-entry*, expressed through the
danger machinery.

**Notes** — the publication is behind a build switch that is always on. A trace of an earlier
version survives as a commented-out condition that would have published it only four times in
five; the randomization was dropped.

## `PostCombatWait`

**Contract** — the few seconds after the last enemy stops being a threat. Does nothing per
cycle; everything is decided at entry.

```text
FUNCTION initialize()
  base.initialize()
  IF I am inside a smart cover THEN RETURN      # the smart cover owns my posture
  movement.gait := run
  posture_goal := IF I just executed a wounded enemy THEN weapon idle
                  ELSE weapon aimed and ready
  IF the item in hand is my best weapon THEN
    weapon_goal(posture_goal, best weapon)
  ELSE IF the item in hand is my sidearm THEN
    weapon_goal(posture_goal, sidearm)
  IF I just executed a wounded enemy THEN RETURN
  IF I remember a last enemy AND it is visible now THEN
    sight := watch it
```

**Invariants** — a creature that has just executed a downed enemy lowers its weapon and does
not stare at the body; one that has just won a firefight keeps the weapon up and watches
where the enemy was. Both behaviours come out of the same branch on one proposition.

**Notes** — the gait is a run, changed at some point from standing still. The action's *own*
duration is the combat planner's post-combat interval — three seconds — during which the
creature is still considered to have an enemy, which is what stops it snapping instantly
back to patrol.

The weapon branch handles only two items: the best weapon and the sidearm. Anything else in
hand is left alone, because the post-combat pose is about lowering a rifle, and a creature
holding something else is in a situation this action does not model.

## `HideFromGrenade`

**Contract** — abandon the whole cover sequence and run to fresh cover chosen against the
grenade, dropping to a crouch on arrival. Fires on the way if the enemy is visible.

```text
FUNCTION initialize()
  base.initialize()
  set property UseSuddenness = false            # the ambush is over
  movement.path_type := level path, smooth detail
  movement.mental_state := danger
  movement.body_state := standing;  movement.gait := run
  invalidate the cover evaluator's current selection
  clear InCover, LookedOut, PositionHolded, EnemyDetoured

FUNCTION execute()
  base.execute()
  IF no danger selected THEN RETURN
  point := best_cover_hint(the danger's position)
  IF point exists THEN setup_cover(point);  movement.gait := run
  ELSE movement.gait := stand still;  movement.body_state := crouch

  IF no enemy THEN sight := watch along the path
  ELSE IF I do not remember the enemy THEN sight := watch along the path; aim_ready()
  ELSE IF the enemy is not visible now THEN
    IF the grenade is within 5 units THEN sight := watch along the path
    ELSE sight := watch the enemy's remembered position
    aim_ready()
  ELSE sight := watch the enemy; fire()

  IF path completed THEN movement.body_state := crouch
```

**Invariants** — the cover hint is asked against the **grenade**, not the enemy. That is the
whole point of the action and the reason it cannot be folded into take-cover.

**Notes** — the five-unit rule is the one piece of real judgment here: with a grenade that
close, the creature stops looking for its enemy and looks where it is running, because it has
about one second and needs to not trip over anything. Further away it keeps tracking the
enemy while it moves.

Dropping to a crouch only on arrival, and immediately standing still if no cover was found,
are the two ends of the same idea: get low, but not while you still need to move.

## `SuddenAttack`

**Contract** — close on an enemy that has not noticed the creature, at a speed and posture
that depend on how near it has got. Ends the moment the enemy sees the creature.

```text
FUNCTION execute()
  base.execute()
  IF no enemy, or I do not remember it THEN
    set property UseSuddenness = false;  RETURN

  IF the enemy is visible now THEN sight := watch it
  ELSE sight := its remembered position, through the look-down guard

  IF the enemy's navigation vertex is somewhere I may go THEN
    steer to that vertex
  ELSE head for the nearest position to it that I may occupy

  distance := my distance to the remembered position
  IF distance >= 15 THEN standing, run
  ELSE IF distance >= 8 THEN standing, walk
  ELSE IF distance >= 4, or the enemy is not visible THEN crouched, run
  ELSE crouched, stand still, and fire()

  IF the enemy is alive AND it cannot see me THEN RETURN
  set property UseSuddenness = false            # I have been noticed; the ambush is over
```

**Invariants** — the ambush ends on *the enemy's* visual memory reporting that it can see
this creature. That is a query against another entity's perception, and it is the only
correct way to model "has he noticed me": asking whether the creature can see the enemy
answers a different question entirely.

**Notes** — the distance ladder is the behaviour. Beyond fifteen units, run upright, because
there is no point sneaking at that range. Between eight and fifteen, walk upright — quieter,
and a walking figure at distance draws less attention than a running one. Under eight,
crouch and *run*, which is the series' signature crouched scuttle. Under four with a clear
view, stop and shoot.

The band from four to six and the band from six to eight are written as separate branches
with identical bodies, which is a leftover from a ladder that once had five rungs.

A group check that would have cancelled the ambush whenever the creature had combat allies
is commented out, so squads may now sneak as a group. A dead-ended path check is similarly
disabled. Both were removals rather than losses; the shipped behaviour is legible without
them.

## `KillEnemyIfPlayerOnThePath`

**Contract** — shoot at the enemy even though the player stands in the line of fire, and
**force the movement layer to update every frame** while doing so.

```text
FUNCTION initialize()
  base.initialize()
  movement.mental_state := danger;  movement.gait := stand still
  movement.force_update := true
  clear InCover, LookedOut, PositionHolded, EnemyDetoured
  play the attack line
  brain.affect_cover := true

FUNCTION execute()
  base.execute()
  IF no enemy THEN RETURN
  sight := watch the enemy;  fire()
  IF I remember the enemy THEN setup_cover(best_cover_hint(remembered position))

FUNCTION finalize()
  base.finalize()
  movement.force_update := false
```

**Invariants** — forcing the movement update is the reason the action exists as a separate
class. The movement layer normally updates at a rate that degrades with distance and load;
with a friendly body between the muzzle and the target, that latency becomes visible as a
stalker shooting the player in the back. Forcing full-rate updates keeps the aim and the
stepping-aside current.

**Notes** — the restoration on exit is unconditional, so a creature that dies in this action
still has the flag cleared.

## `CriticalHit`

**Contract** — the stagger after a critical wound. Stops the creature, lowers its weapon to
idle with a range-appropriate burst shape already selected, fixes the eyes forward, and plays
the injury cry. Per cycle it does nothing; the whole action is its entry and the whole-body
animation that runs alongside it.

**Invariants** — the cover-affecting flag is explicitly cleared, and it is the only action in
the file that clears rather than sets it. A staggering creature is not holding its cover, and
the squad's accounting must know.

**Notes** — the burst shape is selected and handed over even though the goal is *idle*. That
is so the weapon is already configured for the current range when the creature recovers and
the next action issues a fire goal; the stagger does not cost a frame of re-derivation.

The proposition is cleared in exit rather than in execution, so the stagger lasts exactly as
long as the action is selected — which is as long as the whole-body animation blocks
everything else.

## `CombatActionThrowGrenade`

**Contract** — throw a grenade at the enemy, but only once the head is pointed close enough
at it. Remembers which grenade so that its disappearance from the slot means the throw
completed.

```text
FUNCTION initialize()
  base.initialize()
  movement.mental_state := danger
  remember the identity of the grenade in the grenade slot
  movement.gait := stand still;  movement.body_state := standing
  play the grenade-warning line
  set property StartedToThrowGrenade = true

FUNCTION execute()
  base.execute()
  IF the grenade slot is empty or holds a different grenade THEN
    notify the creature the throw completed
    set property StartedToThrowGrenade = false
    RETURN
  IF no enemy, or I do not remember it THEN
    set property StartedToThrowGrenade = false;  RETURN

  IF the enemy is visible now THEN
    target := its actual position and vertex;  sight := watch it
  ELSE
    target := its remembered position and vertex;  sight := watch that position

  IF the angle between my head direction and the direction to the target
     is 22.5 degrees or more THEN RETURN          # keep turning; do not throw yet
  aim the throw at target, naming the enemy as the intended victim
  weapon_goal(FIRE, the grenade, burst shape for the current range)

FUNCTION finalize()
  base.finalize()
  set property StartedToThrowGrenade = false
```

**Invariants** — the throw completes by *the grenade leaving the slot*, not by an animation
callback. Tracking the identity rather than merely the slot's emptiness is what makes the
test correct when the creature has more than one grenade: the next one moving up must not
read as a completed throw.

**Notes** — the aiming cone is the same as the firing cone in the combat action base, and for
the same reason. A grenade released mid-turn lands anywhere.

The warning line plays at entry, before the throw is committed — so a stalker shouts
"grenade" and then may not throw one, if the turn is interrupted. That is the correct
ordering for the enemy, who needs the warning in time to matter.

Naming the enemy as the throw's intended victim, rather than only giving a position, lets the
grenade's own flight logic lead a moving target.

## `CombatActionSmartCover`

**Contract** — fight from an authored smart cover. The action itself does almost nothing:
it flips two flags on the movement layer, points the creature at its cover, and lets the
smart-cover machinery run the loophole state machine.

```text
FUNCTION initialize()
  base.initialize()
  remember the movement layer's "check I can kill the enemy" flag, then set it
  movement.combat_behaviour := true

FUNCTION execute()
  base.execute()
  IF no enemy, or I do not remember it THEN RETURN
  cover := best_cover_hint(the enemy's remembered position)
  IF cover is none THEN
    movement.gait := stand still
    movement.head_for_nearest_accessible_position()
    RETURN
  setup_cover(cover)

  IF the enemy was seen within the last 30 seconds THEN RETURN
  IF the squad allows me to flank, or I am not covering a flank,
     or firing makes no sense THEN RETURN
  set property PositionHolded = true
  set property InCover        = false

FUNCTION finalize()
  base.finalize()
  restore the remembered flag
  movement.combat_behaviour := false
```

**Invariants** — the "check I can kill the enemy" flag makes the smart-cover layer choose
only loopholes that actually bear on the current enemy, rather than any loophole of the
cover. It is saved and restored because a creature may enter combat already inside a smart
cover for a non-combat reason.

**Notes** — the thirty-second timer is the abandonment rule: a creature holding a smart
cover whose enemy has not been seen for half a minute gives up the cover and lets the
sequence advance. It is much longer than the one-to-three-second hold elsewhere, because a
smart cover is an authored strongpoint and leaving one is expensive.

The squad condition attached to that timer is **inverted** relative to the equivalent
condition in `HoldPosition` and in the low-cover hold: those advance the sequence when the
creature *may* flank or is *not* covering one, whereas this one advances only when it may
not and is. Nothing in the source explains the difference, and it is more likely a mistake
than a decision. A rebuild should make the two agree, and should expect the smart-cover
abandonment to behave differently from the original if it does.
