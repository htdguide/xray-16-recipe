# src/xrGame/rat_states.cpp

> The rat's whole behaviour: twelve states, each a short priority-ordered test of the world that either hands the machine to another state or acts.

**Needs** — [`rat_states.h`](rat_states.h.md) · [`rat_state_manager.h`](rat_state_manager.h.md) · [`ai/monsters/rats/ai_rat.h`](ai/monsters/rats/ai_rat.h.md) · [`ai/monsters/ai_monster_squad_manager.h`](ai/monsters/ai_monster_squad_manager.h.md) · [`ai/monsters/ai_monster_squad.h`](ai/monsters/ai_monster_squad.h.md) · [`entity_alive.h`](entity_alive.h.md)
**Used by** — reached through its declarations in [`rat_states.h`](rat_states.h.md); callers name that, not this file.
**Tier floor** — T3: a dozen boolean tests per rat per update

## Purpose

Rats are the engine's cheapest creature and they deliberately skip the goal/plan machinery
every other creature uses. Instead each behaviour is a hand-written run of guards, checked
in a fixed order, that either transitions the pushdown machine or performs this update's
action. The **order of the guards is the behaviour** — it is the priority ordering of the
rat's concerns, and it is the only place that ordering is written down.

This file is a separate compilation unit from the rat itself only because the rat is large;
a rebuild may fold it in. What must not be folded is the distinction between the machine
([`rat_state_manager.cpp`](rat_state_manager.cpp.md)) and the states, because the machine's
enter/run/leave discipline is what the states are written against.

## State

`Stateless.` Every state is a behaviour object with no fields beyond the rat back-reference
the base interface supplies. All mutable data lives on the rat.

## The three transition verbs

Every state reaches its machine through exactly three operations, and which one a state
picks is a decision, not a detail:

- **push** — layer a state on top, intending to come back. Used for interruptions: an enemy
  appears, morale breaks, a noise startles the rat. The interrupted state resumes when the
  interruption pops.
- **pop** — the reason for this state is gone; resume whatever wanted it. Used when the
  enemy is lost, the timer expires, morale recovers.
- **change** — replace the top without growing the stack. Used when this state was simply
  the *wrong* state — the rat's own classifier disagrees with where it is.

A state that transitions must return immediately afterward: the machine runs `execute` for
the *current* state, so anything after the transition call still belongs to the outgoing
state. Every state here obeys that.

## The common preamble

**Contract** — every state except `death` begins by returning without acting if the rat is
not alive, and nine of the twelve then check whether a patrol path is in force and divert
to `no way`. These two guards are the invariant the whole file rests on: **behaviour only
runs for a live rat, and an authored path outranks every behavioural decision.**

```text
FUNCTION execute()             # the shape shared by every live-rat state
  IF NOT rat.alive THEN RETURN
  IF rat.walking_a_path THEN
    machine.push(no_way)       # or change(no_way) where the state should not resume
    RETURN
  ... this state's own guards ...
```

**Notes** — some states *push* `no_way` and some *change* to it. The difference is whether
the state expects to be resumed once the path segment is walked: roaming and pursuing do,
passive roaming and the flinch do not. This looks arbitrary and largely is; a rebuild
choosing one uniformly will change which state a rat returns to after a patrol segment, but
not obviously for the worse.

## The world predicates

The states are written against a small vocabulary of questions about the rat's world,
answered by the rat. They are named here once so the state descriptions can be terse:

```text
enemy            : a memory-selected enemy exists at all
enemy_alive      : that enemy is still alive
enemy_lost       : perception of the enemy has aged past the give-up window
beyond_pursuit   : the enemy is further from the rat's HOME than the pursuit radius
                   # note: measured from home, not from the rat — this is what
                   #   tethers a rat to its nest instead of to its current position
at_home          : the rat is within the home radius of its home position
out_of_reach     : the enemy is further than the attack distance
off_angle        : the rat's body yaw differs from its aim yaw by more than the
                   attack angle
in_range_aimed   : close enough and pointed at the enemy
in_range_unaimed : close enough but not pointed at it
morale_ok        : morale has recovered to its normal value
food             : a memory-selected item to eat exists
startled         : a sound arrived since the last update, from nobody on this rat's
                   team, and not from the thing it is eating, and no enemy is selected
recoil_expired   : the flinch has lasted past its fixed two-second window
attack_ready     : the per-rat attack cooldown has elapsed
```

**Notes** — `beyond_pursuit` measuring from the home position rather than from the rat is
the single most consequential choice in the file: it is why a rat colony defends a place
rather than chasing individuals across a level, and why the attack states keep re-checking
it rather than committing to a target.

## `death`

**Contract** — the terminal state. Does nothing at all while the rat still carries food
value; once that is exhausted it disables the rat and sends a destroy event for its own
identifier, which is what actually removes the entity.

```text
FUNCTION execute()
  IF rat.food_value > 0 THEN RETURN     # the corpse is still worth eating; leave it
  rat.enabled = false
  send_destroy_event(rat.id)
```

**Notes** — the corpse persists exactly as long as it is a resource for other rats. This is
the whole of the rat corpse lifetime: there is no timer. A rebuild that adds one changes
how long rat carcasses litter a level.

## `free active`

**Contract** — the default roaming state and the only one that reads the *full* set of
concerns. On entry it seeds the roam unless a path is already in force. Its guards, in
priority order, are the rat's value system:

```text
FUNCTION execute()
  common preamble, pushing no_way
  IF enemy AND (NOT beyond_pursuit OR NOT at_home) THEN push(attack_melee); RETURN
  IF NOT morale_ok                                 THEN push(under_fire);   RETURN
  IF startled                                      THEN push(free_recoil);  RETURN
  IF food                                          THEN push(eat_corpse);   RETURN
  rat.roam_actively()
```

**Invariants** — enemy outranks fear outranks noise outranks hunger. The enemy guard's
second clause is the tether: a rat attacks an enemy that is inside the pursuit radius of
home, *or* an enemy of any distance while the rat itself is already away from home. A rat
standing on its nest will not leave it for a distant target.

## `free passive`

**Contract** — roam without reacting to anything. No enemy, morale, noise or food guard:
only the alive check and the path diversion, and the path diversion *replaces* this state
rather than layering on it. The state a rat is left in when something else wants it inert.

## `attack range`

**Contract** — hold position and shoot. Pops when the enemy is gone. Diverts to `no way`
when the rat cannot hold this spot: either it is not mid-reposition and the spot is not
standable, or the enemy has moved out of attack distance, or the rat's body has swung off
the attack angle. Otherwise it runs the ranged attack. On leaving, **it stops firing** —
the only `finalize` in the file that undoes something, and the reason a rat does not keep
shooting into a state it has left.

```text
FUNCTION execute()
  IF NOT rat.alive THEN RETURN
  IF NOT enemy THEN pop(); RETURN
  IF (NOT rebuilding_position AND NOT can_stand_here) OR out_of_reach OR off_angle THEN
    push(no_way); RETURN
  rat.attack_at_range()

FUNCTION finalize()
  rat.stop_firing()
```

**Notes** — `no way` is doing double duty here. It is named for "a path is in force" but it
is also the rat's reposition state, entered whenever the current spot has stopped working.
A rebuild would name it *reposition* and treat the authored-path case as one of its inputs.

## `attack melee`

**Contract** — the busiest state: closing on the enemy, and also where a rat squad elects
its leader. Its guards in order: if the rat's own classifier no longer says melee, hand to
whatever it does say; if there is an enemy but it is beyond the pursuit radius, go home; if
there is no enemy, pop. Then, if the enemy is close and the rat is aimed, escalate to
ranged attack; if close but unaimed, turn and keep moving without changing state.

Past the guards comes squad bookkeeping, and it is the load-bearing part:

```text
FUNCTION execute()
  IF NOT rat.alive THEN RETURN
  IF rat.classify_state() != attack_melee THEN change(rat.classify_state()); RETURN
  IF enemy THEN
    IF beyond_pursuit THEN change(return_home); RETURN
  ELSE
    pop(); RETURN
  IF in_range_aimed   THEN change(attack_range); RETURN
  IF in_range_unaimed THEN rat.turn_toward_enemy(); rat.move(); RETURN

  squad = squad_of(rat)
  IF squad EXISTS THEN
    # Leadership is claimed, not granted: a rat promotes itself when the
    # current leader is dead, or when it finds it is not a member at all.
    IF (squad.leader != rat AND NOT squad.leader.alive) OR squad.index_of(rat) IS none THEN
      squad.leader = rat
    # The leader re-assigns every member a slot around the enemy, but only when
    # the squad's live count has changed since it last did so. Re-slotting is
    # expensive and only a death or an arrival can invalidate the assignment.
    IF squad.active AND squad.leader == rat AND rat.known_squad_size != squad.live_count THEN
      squad.assign_attack_slots(rat.enemy)
      rat.known_squad_size = squad.live_count
  rat.face_assigned_direction()
  rat.move()
```

**Invariants** — the squad-size comparison is the *only* thing that triggers re-slotting.
If a rebuild recomputes slots every update, a pack of rats will thrash between positions;
if it never recomputes, dead members leave holes in the encirclement.

**Notes** — self-election means leadership can be claimed simultaneously by several rats in
one update, with the last writer winning. Harmless, because the next update re-checks, but
it is why there is no election protocol here.

## `under fire`

**Contract** — the morale-broken state. Entered by push, so it resumes whatever was
happening. On entry it picks a retreat target. Then: an enemy appearing replaces this state
with melee outright. Otherwise, a fresh sound that came from an entity escalates to melee
by *push* — the rat turns on whatever startled it — and a sound from nowhere just refreshes
the query timestamp. Recovered morale pops. Anything else, keep moving away.

```text
FUNCTION execute()
  IF NOT rat.alive THEN RETURN
  IF enemy THEN change(attack_melee); RETURN
  IF sound_arrived_since_last_update THEN
    IF the sound had an emitter THEN push(attack_melee); RETURN
    rat.mark_sound_query_time()
  IF morale_ok THEN pop(); RETURN
  rat.move()
```

**Invariants** — the enemy path *changes* and the sound path *pushes*. A rat that turns on
a noise and finds nothing returns to being afraid; a rat that acquires a real enemy stops
being afraid entirely. That asymmetry is the state's entire point.

## `retreat`

**Contract** — withdrawing while the enemy is still live. Pops when the enemy is confirmed
gone. While there is a live enemy: defer to the rat's own classifier if it disagrees;
escalate to ranged attack if close and aimed; just move if close and unaimed; otherwise
re-pick a retreat point *away from the enemy*. Then face the retreat direction and move.

**Notes** — nothing in the shipped state registration pushes this state. It is reachable
only through the rat's classifier disagreeing inside another state, which makes it the
file's most likely dead code. Treat its existence as informative, not required.

## `pursuit`

**Contract** — chasing a remembered enemy. Gives up (pop) when perception of the enemy has
aged out; escalates to melee (push) when the enemy is perceived again; yields to fear
(push `under_fire`) when morale is gone; yields to a startle by *changing* to the flinch.
Otherwise it heads for the enemy's last known position.

**Invariants** — the give-up test is first. A rat that has lost the trail stops chasing
before it considers anything else, which is what prevents a pursuit from outliving the
memory that justified it.

## `free recoil`

**Contract** — the flinch. On entry, unless a path is in force, it seeds the recoil and
picks a point *behind* the rat to back toward. Pops on an enemy appearing, pops when the
two-second window expires, and would change to pursuit on a lost-contact timer — a branch
guarded by the same enemy test that already popped, so it is unreachable. Otherwise it
backs away.

**Notes** — the unreachable pursuit branch is a real bug preserved here because a rebuild
reading only the state list would otherwise expect the flinch to lead into a chase. It does
not; the flinch always ends by popping.

## `return home`

**Contract** — walking back inside the home radius. Its guard order encodes the tether's
release condition: an enemy that is *not* beyond the pursuit radius resets the goal timer
and escalates to melee; an enemy that is close is engaged in place (unaimed: do nothing
this update; aimed: change to ranged attack); and the state pops when there is no enemy at
all, or the rat has arrived home, or the enemy has died. Otherwise it aims at home and
walks.

```text
FUNCTION execute()
  common preamble, pushing no_way
  IF enemy AND NOT beyond_pursuit THEN rat.reset_goal_timer(); push(attack_melee); RETURN
  IF enemy THEN
    IF in_range_unaimed THEN RETURN                   # stand and let the turn resolve
    IF in_range_aimed   THEN change(attack_range); RETURN
  IF NOT enemy OR at_home OR NOT enemy_alive THEN pop(); RETURN
  rat.aim_at_home(); rat.walk_home()
```

**Invariants** — a rat going home will still fight something that gets close enough,
because the in-range tests sit *above* the arrival test. Without that ordering a fleeing
rat would be free damage.

## `eat corpse`

**Contract** — feeding. Pops when the situation no longer permits eating: an enemy that is
beyond the pursuit radius, or the rat being away from home, or (with no enemy) broken
morale. Then it extends the goal timer by ten seconds — feeding is time-boxed and the box
is renewed every update the rat keeps eating — yields to a startle, and otherwise eats. On
leaving, it re-arms the rat's firing.

**Notes** — the ten-second renewal is the only way a rat's eating ends on its own: the goal
timer is what eventually expires, and re-setting it every update means eating continues
until an external condition breaks it. Whether ten is tuned or arbitrary is not recoverable
from the source; nothing else uses that number.

## `no way`

**Contract** — two behaviours under one name, selected by whether a patrol path is actually
in force. On entry it stamps the attack-cooldown clock.

- **No path in force** — this is the *reposition* case, reached from the attack states. If
  the attack cooldown has elapsed, push melee and resume attacking. Otherwise pick a point
  behind the rat, face it and move: the rat backs off for the remainder of its cooldown.
- **A path in force** — advance to the next way point and adopt that point's movement
  flags (whether speed may be adjusted, whether to go straight at it), then move. The flags
  come from the authored path, which is why they are read per way point rather than set
  once.

```text
FUNCTION execute()
  IF NOT rat.alive THEN RETURN
  IF NOT rat.walking_a_path THEN
    IF cooldown_elapsed THEN push(attack_melee); RETURN
    rat.pick_point_behind(); rat.face_it(); rat.move()
  ELSE
    rat.advance_way_point()
    rat.set_movement_mode(point.can_adjust_speed, point.straight_forward)
    rat.move_along_path()
```

**Notes** — merging "an author told this rat where to walk" with "this rat's firing spot
stopped working" into one state is the least defensible decision in the file. A rebuild
should split it; nothing outside this file depends on the two being the same identifier
beyond the transitions listed above.
