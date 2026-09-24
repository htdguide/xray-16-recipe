# src/xrGame/ai/monsters/poltergeist/poltergeist_state_attack_hidden_inline.h

> The poltergeist's attack, which is a movement pattern and nothing else: orbit the target at an authored radius, reversing direction on a timer, shrinking the orbit when the level will not accommodate it and growing it back when it will.

**Needs** — [`poltergeist_state_attack_hidden.h`](poltergeist_state_attack_hidden.h.md) · [`poltergeist.h`](poltergeist.h.md) · `../states/monster_state_attack_move_to_home_point.h` · [`../monster_sound_defs.h`](../monster_sound_defs.h.md) · [`../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md)
**Used by** — [`poltergeist_state_attack_hidden.h`](poltergeist_state_attack_hidden.h.md)
**Tier floor** — T3: sampling a circle against the navigation graph

## Purpose

The poltergeist never closes on its target. Its attack state's entire job is to keep the
creature **orbiting the target at a fixed radius**, because the flame and telekinesis abilities
both fire on their own schedules from wherever the creature happens to be, and both require it
to be within their engagement distance. Orbiting keeps it in range without ever letting it be
cornered or caught.

Three decisions shape the orbit and each is visible in play:

**Direction is re-picked on a timer, not maintained.** Every so often the creature works out
which side of the target it is on relative to the target's facing, and heads that way — then
flips a coin and reverses it half the time. The result is an orbit that changes direction
unpredictably but not constantly, which is what makes the creature hard to track and hard to
lead a shot on.

**The radius breathes.** The desired point is sought by sampling twelve positions around the
circle in the chosen direction, taking the first that lies on the navigation graph. When all
twelve fail — a tight room, a corridor — the radius shrinks a tenth and the state gives up for
the tick. When one succeeds it grows a tenth back, capped at the authored radius. So the
creature settles into the largest orbit the room allows, automatically.

**A route is rebuilt every fifth of a second.** The target point is recomputed each tick and
the route builder is told to re-plan on a very short period, because the point moves with the
target and a stale route means the creature flies where the target *was*.

## State

```text
RECORD AttackHidden
  reverse_at      : int    # global clock at which the orbit direction is reconsidered
  radius_fraction : real   # invariant: within [0.1, 1.0]
  going_left      : bool
  target          : vector
  target_vertex   : int
```

## `enter`

**Contract** — prepares the creature's route builder, clears the direction timer so a direction
is chosen on the first tick, and resets the radius fraction to full — every fresh engagement
starts by trying the widest orbit.

## `run`

**Contract** — the per-tick step. First gives the return-home child a chance to take over;
otherwise picks an orbit point, points the route builder at it, and sets the animation and
sound. Does not run a sub-state in the orbiting case.

```text
FUNCTION run()
  IF home_claim()
    select move_to_home_point
    current_state.run()
    previous_substate = current_substate
    RETURN

  current_substate = none ; previous_substate = none    # orbiting is not a sub-state

  pick_orbit_point()

  route.target        = (target, target_vertex)
  route.rebuild_every = 200 ms
  route.stop_within   = 3 units
  route.use_covers    = false              # a floating creature does not take cover

  animation.action     = run
  animation.accelerate = aggressive
  animation.braking    = off
  play the aggressive vocalisation on the creature's attack-sound cooldown
```

**Invariants** — when orbiting, both the current and previous sub-state are cleared to *none*,
so the return-home child's start-or-continue test always sees "was not running last tick" and
asks its start condition rather than its completion. That is the only reason the two branches
compose: without the clearing, one tick of orbiting would leave the home child looking as
though it were still in progress.

**Notes** — `stop_within` of three units is what stops the creature landing exactly on the
orbit point and then having nowhere to go; it treats arrival generously and picks a new point
next tick.

The rebuild period of 200 milliseconds is short enough that the route effectively tracks a
walking target and long enough to be affordable. It is the one number in this file with an
obvious cost/quality reading, and it is not derived.

Braking is disabled and acceleration is set aggressive, so the creature is always at its top
gait — a poltergeist has no cruising speed.

## `pick_orbit_point`

**Contract** — chooses the next point on the orbit. Pure except for updating the orbit state.

```text
FUNCTION pick_orbit_point()
  target_pos  = the enemy's position
  self_pos    = the creature's position
  self2enemy  = target_pos - self_pos
  radius      = creature.fly_around_distance * radius_fraction

  # --- reconsider which way round, on a timer --------------------------
  IF now > reverse_at
    front_point = target_pos + the enemy's own facing scaled by radius
    left_side   = self2enemy CROSS (front_point - self_pos) is positive in the horizontal plane
    IF a coin flip comes up heads THEN left_side = NOT left_side
    going_left = left_side
    reverse_at = now + creature.fly_around_change_direction_time seconds

  # --- sample the circle in that direction -----------------------------
  from_enemy = the direction from the enemy back to the creature, scaled to radius

  FOR step FROM 1 TO 12
    angle     = step * (a full turn / 12), negated if going_left
    candidate = target_pos + from_enemy rotated by angle

    IF candidate lies on the navigation graph
      target        = candidate
      target_vertex = the vertex at candidate
      radius_fraction = min(radius_fraction + 0.1, 1.0)     # room to breathe: grow back
      RETURN

  # nothing on the circle was navigable
  radius_fraction = max(radius_fraction - 0.1, 0.1)         # tighten and retry next tick
  target          = self_pos                                # stand still this tick
  target_vertex   = the creature's own vertex
```

**Invariants** — the radius fraction moves by a tenth per tick in either direction and is
clamped to `[0.1, 1.0]`, so the orbit can collapse to a tenth of its authored size but never
further, and recovers at the same rate. Because it grows on *every* success, a creature in open
space is always at the full radius within ten ticks of leaving a tight space.

**Notes** — the direction decision starts from which side of the target the creature is on
*relative to where the target is facing*, computed as a horizontal cross product against a
point one radius in front of the target. That bias means the creature tends to circle towards
the target's front — it wants to be seen, which suits a creature whose whole threat is
presence. The coin flip then discards that bias half the time, which is what prevents a player
from predicting the orbit by turning.

Sampling starts at one step off the creature's current bearing and walks around, so the nearest
legal point in the chosen direction wins and the creature moves a short arc rather than jumping
across the circle.

Twelve samples, thirty degrees apart, and a step of a tenth on the radius, are unexplained.

The failure case sets the target to where the creature already is, which through the route
builder reads as "stop", and the state then relies on the shrunken radius to succeed next tick.
The creature visibly hesitates in a doorway before tightening its orbit; that is this branch.

## `home_claim`

**Contract** — the start-or-continue rule applied to the single registered child, the state
that flies the creature back inside its home. True means the child should run this tick instead
of the orbit.

**Notes** — this is the only thing that can interrupt the attack. A poltergeist with an enemy
outside its home orbits until the home state's start condition fires, then returns — which,
combined with `run_home_point_when_enemy_inaccessible` answering *no* on the creature, means a
poltergeist gives up on a target because of *where it itself is*, never because of where the
target is.
