# src/xrGame/ai/monsters/rats/ai_rat_templates.cpp

> The rat's own locomotion: it integrates a heading and a position every frame, asks the navigation mesh whether the step it just computed is legal, and backs the whole step out when it is not. This is what the rest of the game's creatures delegate to the path builder, and the rat does not.

**Needs** — [`ai_rat.h`](ai_rat.h.md) · [`../../../../xrAICore/Navigation/level_graph.h`](../../../../xrAICore/Navigation/level_graph.h.md) · [`../../../../xrAICore/Navigation/game_graph.h`](../../../../xrAICore/Navigation/game_graph.h.md) · [`../../../../xrAICore/Navigation/PatrolPath/patrol_path.h`](../../../../xrAICore/Navigation/PatrolPath/patrol_path.h.md) · [`../../../location_manager.h`](../../../location_manager.h.md) · [`../../../movement_manager.h`](../../../movement_manager.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [Seam: Static collision database](../../../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: per-frame vector integration with a mesh-membership test in the inner loop

## Purpose

The rat has no path to follow. It has a *goal point*, a heading it steers toward that point,
and a mesh it may not leave. Movement is an integration, and legality is a rejection test
applied after the fact — propose the step, ask the mesh, take it or undo it. Four consequences
that a rebuild must reproduce or knowingly discard:

- A rat has **no route**. It cannot go around anything, and it does not know a wall is there
  until it has tried to walk into it.
- Being blocked is expressed as a **flag plus a saved previous position**, not as a failed
  request. The recovery is "turn a hundred and eighty degrees and pick a goal behind me".
- Speed is not continuous. It is one of four authored values, chosen by the angle between where
  the rat is facing and where it wants to go — which is why the rat *slows down to turn* and why
  the animation and pitch-rate lookups elsewhere can compare speeds for equality.
- Mesh membership is checked **twice** per step, once as a lookahead during speed selection and
  once as the veto on the actual move. The first is what usually keeps the rat off the edge; the
  second is what catches it when the first was not enough.

## `move` — the step

**Contract** — advances the rat by one frame. Takes two flags: whether it may adjust its own
speed, and whether it is travelling straight at its goal rather than wandering toward it. Writes
the rat's transform and orientation, or restores them unchanged. Does not allocate.

```text
FUNCTION move(may_adjust_speed, straight_at_goal)
  save orientation, position, body target, and turn rate

  IF may_adjust_speed
    select_speed()                     # may also rewrite the body's target heading

  IF turn_needed > 30 degrees
    make_turn()                        # stop dead and rotate; no translation this frame
    RETURN

  IF speed IS zero
    RETURN

  proposed = IF blocked_last_frame THEN old_position ELSE calc_position()

  IF calc_node(proposed)               # the mesh accepts it
    commit position and orientation
    old_position   = the position we started this frame at
    blocked        = false
  ELSE
    speed          = epsilon           # not zero: zero would skip the whole step next frame
    blocked        = true
    restore orientation, position, body target and turn rate

  IF blocked AND (not mid-turn OR the turn has finished)
    IF now - last_reverse > 500 ms
      body target heading = current heading + half a turn
      offset = unit vector along the new heading, times 100 if straight_at_goal
      IF NOT following a patrol path
        goal = my position + offset            # the new goal is behind me
      last_reverse = now
    IF NOT following a patrol path
      make_turn()
```

**Invariants**

- **The whole step is transactional.** Orientation, position, body target and turn rate are all
  saved before anything is touched and all restored together on rejection. Restoring only the
  position — the obvious partial fix — leaves the rat facing into the wall and it re-proposes
  the same step forever.
- **On rejection the speed becomes an epsilon, not zero.** A zero speed returns early on the
  *next* frame before the recovery below can run, so the rat would be permanently stuck. This is
  the least obvious line in the file and the one a rebuild is most likely to get wrong.
- **The reversal is rate-limited to twice a second.** Without the limit a rat wedged in a corner
  flips direction every frame and vibrates in place.
- **Turning is a stop, not a steer.** Any required turn beyond thirty degrees translates the rat
  not at all this frame. That threshold is twice the fifteen degrees the animation selector uses
  to decide to play a turn clip, so there is a band where the rat is turning-while-moving with a
  turn animation — deliberate, and what stops the turn from reading as a snap.
- **A patrolling rat is exempt from the reversal.** Following an authored path, it neither
  re-goals nor turns around when blocked; it keeps leaning into the obstruction until the path's
  next waypoint pulls it clear. That is what keeps an authored route authored.
- **The hundred-fold multiplier** on the reversal offset applies only when travelling straight at
  a goal, which puts the new goal effectively at infinity behind the rat — a direction rather
  than a destination. Wandering rats get a goal one unit behind them instead, which they reach
  almost at once and then re-roll.

## `select_speed` — the speed ladder

**Contract** — chooses the rat's speed and turn rate from the angle between its facing and its
goal, then lowers both if a lookahead at that speed would leave the mesh. May rewrite the body's
target heading. Called only when the caller permits speed adjustment.

```text
FUNCTION select_speed()
  angle = angle between my facing and the direction to my goal

  CASE the speed I am currently at OF
    min_speed:
      IF angle >= 120 deg   speed = 0;         turn_rate = standing;  face the goal
      ELSE                  speed = min_speed; turn_rate = min
    max_speed:
      IF angle >= 120 deg   speed = 0;         turn_rate = standing;  face the goal
      ELSE IF angle >= 90   speed = min_speed; turn_rate = min
      ELSE                  speed = max_speed; turn_rate = max
    attack_speed:
      IF angle >= 90        speed = min_speed; turn_rate = min
      ELSE IF angle >= 45   speed = max_speed; turn_rate = max
      ELSE                  speed = attack_speed; turn_rate = attack
    anything else:
      face the goal; speed = 0; turn_rate = standing

  # lookahead: one frame at the chosen speed, straight ahead
  IF that point is outside the mesh
    IF I was at attack speed AND a max-speed step would be inside
      speed = max_speed; turn_rate = max
    ELSE
      speed = min_speed; turn_rate = min
```

**Invariants**

- **The ladder is asymmetric on purpose.** A rat at attack speed never stops to turn — its worst
  case is dropping to minimum — while a wandering rat does. A charging rat that froze to turn
  would be trivially avoidable.
- **Each speed has its own turn rate**, and the pairing is what makes turning radius scale with
  speed rather than a fast rat pivoting on the spot.
- **The lookahead is one frame, straight ahead, at the chosen speed.** It is not a probe of the
  route — it cannot be, there is no route — and its only effect is to slow the rat near an edge
  so the hard veto in `move` fires less often. The attack-speed special case degrades by one
  rung rather than two, again so that a charge is not cancelled by a lip in the floor.
- The thresholds (120, 90, 45 degrees) are compiled in and have no recoverable derivation beyond
  being a monotone ladder.

## `calc_position` — the heading integration

**Contract** — computes the position the rat would occupy after one frame, and updates its
integrated heading and pitch as a side effect. Does not commit anything.

```text
FUNCTION calc_position() -> vector
  turn_step = turn_rate * frame_elapsed
  to_goal   = goal - my position

  IF NOT straight_at_goal
    # pitch tracks the goal's height with a dead band and a damper
    IF to_goal.height >  1     pitch = min(pitch + turn_step,  0.8)
    ELSE IF to_goal.height < -1 pitch = max(pitch - turn_step, -0.8)
    ELSE                        pitch = pitch * 0.95

  alignment = dot(my facing, normalize(to_goal))         # 1 = dead ahead
  urgency   = (1 - alignment) / 2 * turn_step * 10       # how hard to turn this frame
  side      = cross(normalize(to_goal), my facing)       # which way

  IF straight_at_goal
    # a dead band: nearly aligned means stop turning, roughly aligned means a fixed nudge
    IF alignment > 0.95        heading_rate = 0
    ELSE IF alignment > 0.75   heading_rate = ±0.10
    heading_rate = (heading_rate * 9 + ±urgency) * 0.1   # then the same exponential filter
  ELSE
    heading_rate = (heading_rate * 9 + ±urgency) * 0.1

  heading = heading + heading_rate

  # pitch is NOT integrated: it is re-derived from the mesh cell the rat stands on
  plane = the plane of my current mesh cell's triangle
  project my position and a point one unit ahead onto that plane
  pitch = -(the pitch of the vector between those projections)

  heading = normalize_signed(heading)
  RETURN my position + my facing * speed * frame_elapsed
```

**Invariants**

- **The heading rate is exponentially filtered with a nine-to-one weighting**, so the rat's turn
  rate ramps rather than snapping. This is the single line that makes rat movement look organic
  rather than mechanical, and its time constant is roughly ten frames.
- **The `straight_at_goal` dead band exists to stop a charging rat from weaving.** Without it,
  the urgency term never reaches zero and a rat running at a target oscillates around the line
  to it. The two thresholds are the only difference between wandering steering and charging
  steering.
- **Pitch is computed twice and only the second computation survives.** The first — the damped
  tracking of the goal's height — is written into the same field the mesh-derived pitch then
  overwrites. The dead-band tracking is therefore dead code, which is worth noting because it
  looks like the load-bearing part and is not: **the rat's pitch is the slope of the floor it is
  standing on**, full stop, which is what makes it run up ramps convincingly.
- The translation uses the rat's *facing*, not the direction to the goal, so the rat travels
  along its nose and the goal only influences it through the heading integration. That is why it
  arcs into a goal rather than sliding toward it.

## `calc_node` — the mesh veto

**Contract** — asks whether a proposed position is legal: it must resolve to a valid mesh
vertex, that vertex must contain the position, and the position must be inside the rat's
movement restrictions — *or* the rat must already be outside them. Rewrites the proposed
position's height to the mesh cell's surface as a side effect. Returns the verdict.

**Invariants** — the "or I am already outside" clause is the escape hatch: a rat that somehow
ends up in forbidden space would otherwise have every step rejected and be frozen forever. With
it, a rat outside its restriction can move freely until it re-enters — and then is held.

Only the *current* vertex is tried first, falling back to a neighbourhood search; that ordering
is the optimisation that makes a per-frame test affordable for dozens of rats.

## `bfCheckIfOutsideAIMap`

**Contract** — the lookahead form of the same question, without the restriction check and
without rewriting anything. Used only by `select_speed`.

## `make_turn`

**Contract** — stops the rat dead and rotates its transform to its integrated heading and pitch,
preserving its position exactly. Sets the body's turn rate to one full turn per second for the
duration.

**Invariants** — biting suppresses the turn entirely when the rat is already within thirty
degrees of its target, so a rat mid-bite does not pivot away from what it is biting.

The position is saved and restored around the rotation because setting the transform's
orientation clears its translation — an artefact of the matrix helper, not a decision.

## `select_next_home_position` — the nest migration

**Contract** — chooses the next coarse cross-level vertex the nest will drift toward, and
re-arms the migration clock. Called only when the clock has expired and the nest has arrived.

```text
FUNCTION select_next_home_position()
  candidates = neighbours of next_graph_point whose terrain type I may occupy
  preferred  = those candidates excluding the one I came from

  IF preferred IS empty
    pick the first admissible neighbour, including the one I came from  # a dead end: turn back
  ELSE
    pick uniformly among preferred                                      # never immediately backtrack

  current_graph_point  = next_graph_point
  next_graph_point     = the choice
  graph_point_change_at = now + uniform(60_000 .. 120_000)
```

**Invariants** — **excluding the vertex the nest came from is the whole of the migration's
character**: without it the nest random-walks and stays put, with it the nest travels. The
fallback to the excluded vertex handles a dead end, where turning back is the only option.

**Notes** — the one-to-two-minute re-arm appears here and again at spawn, and is compiled in
both times with no derivation.

## `get_next_target_point` — following an authored path

**Contract** — returns the world position of the patrol waypoint the rat is heading for,
advancing to the next one (wrapping at the end) when the rat comes within one and a half units
of the current one. Disables patrolling entirely if the path or the index is missing, returning
the rat's own position so it stands still for one tick rather than steering at nothing.

**Invariants** — reaching a waypoint also **elects this rat squad leader**. That is how an
authored path drags a whole nest: the rat on the path becomes the leader, the leader's position
becomes the home anchor, and every other rat's anchor follows it. A rebuild that patrols without
the election gets one rat walking a route and a nest that ignores it.

The arrival radius of one and a half units is compiled in and is what stops a rat from orbiting
a waypoint it cannot stand exactly on.

## `can_stand_here` and `can_stand_in_position`

**Contract** — both ask whether the rat's oriented bounding box overlaps another *rat's*, using a
spatial query for candidates and an oriented-box intersection to decide. They differ only in
the query radius: `can_stand_here` uses the rat's own radius, `can_stand_in_position` a fixed
fifth of a unit.

**Invariants** — only other rats are considered; the test is about rats not standing inside each
other, not about collision with the world, which the mesh veto already handles. It is what stops
a settling rat from freezing on top of a neighbour: a rat may only leave the activity budget
when it can stand where it is.

**Notes** — the two routines are otherwise identical, and the fixed radius in the tighter one is
a magic number with no derivation. They could be one routine with a radius argument.

## `fire`, `movement_type`, `set_firing`, `set_position`, `set_pitch`

**Contract** — small setters the states use: arm or disarm the bite (setting both the flag and
the pending action), assign a speed and derive the moving flag from it, write the transform from
the integrated orientation and a position, and push the integrated orientation into the body's
target so the animation layer sees it.

## `draw_way`

**Contract** — developer builds only: draws the rat's patrol path as a closed loop of lines with
a marker at each waypoint. No effect on behaviour.
