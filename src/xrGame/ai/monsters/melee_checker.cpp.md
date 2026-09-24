# src/xrGame/ai/monsters/melee_checker.cpp

> Decides when a creature is close enough to swing, using a head-to-target measurement rather than a centre-to-centre one, and narrowing or widening its own idea of "close enough" based on whether recent swings actually connected.

**Needs** — [`melee_checker.h`](melee_checker.h.md) · [`melee_checker_inline.h`](melee_checker_inline.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`monster_enemy_manager.h`](monster_enemy_manager.h.md) · [Seam: Static collision database](../../../../SYSTEM-REQUIREMENTS.md#seam-static-collision-database)
**Used by** — [`melee_checker.h`](melee_checker.h.md)
**Tier floor** — T2: a ray query per tick against the collision database, with a reused result buffer

## Purpose

Creature melee reads badly when range is judged centre to centre: a creature with a long body
lunges from too far, and a creature standing above or below its target swings at nothing. Two
decisions fix that here.

First, the range measured is *head to target centre*, and where the target is within a few
metres the straight-line distance is replaced by the range along a ray cast from the head. The
ray answers "how far would a swing have to reach", which is what the animation actually does,
and it only counts when the first thing the ray hits *is the target* — an obstacle between them
leaves the straight-line distance in place rather than reporting a shorter one.

Second, the inner edge of the melee window is **not** a constant. It adapts: two consecutive
connecting swings widen it (the creature gets bolder about starting from further out), two
consecutive misses narrow it (it insists on closing further before it commits). The outer edge
moves with it by the same offset, so the window's width is constant and only its placement
slides.

## State

```text
RECORD MeleeChecker
  creature            : BaseMonster
  ray_results         : ray_result_buffer   # reused across calls; never freed per call

  # authored in the creature's configuration section — see melee_checker_inline.h
  min_attack_distance : real                # the widest the inner edge may become
  max_attack_distance : real                # the outer edge at full confidence
  floor_distance      : real                # the narrowest the inner edge may become
  adapt_step          : real                # how far one confirmed run moves the edge

  swing_history       : list<bool>          # fixed length 2, newest first
  current_min         : real                # the live inner edge
```

Invariants: `current_min` stays within `[floor_distance, min_attack_distance]` — both ends
snap rather than overshoot. `swing_history` always holds exactly two entries; there is no
"not yet swung" value, so `begin_attack` must seed it.

The window width is an invariant of the arithmetic, not a stored field: the outer edge is
always `max_attack_distance` shifted by the same amount the inner edge has been shifted from
`min_attack_distance`.

## `distance_to_enemy`

**Contract** — returns the distance the creature should treat as its reach to the given
target. Does not allocate (the result buffer is a field, cleared and refilled). Casts at most
one ray, and only when the target is already close.

```text
FUNCTION distance_to_enemy(target) -> real
  distance = straight_line(creature.position, target.position)
  IF distance > MAX_TRACE_RANGE
    RETURN distance                  # too far to be worth a ray; the number is only a gate

  head  = creature.head_position()
  aim   = direction_from(head, target.centre())

  cast one ray from head along aim, length MAX_TRACE_RANGE,
      against dynamic objects only, nearest hit only, back-faces culled
  IF a hit came back AND the nearest hit IS the target
    distance = range of that hit     # the reach a swing would actually need

  RETURN distance
```

**Notes** — the ray range cap is six world units. It is the only unexplained constant here: it
is comfortably larger than any shipped creature's melee window, so it works as "close enough
that the correction matters", but nothing derives it.

The query asks for *objects*, not static geometry, so a wall between creature and target does
not shorten the answer — it simply fails to name the target as the nearest hit, and the
straight-line distance stands. That is deliberate: the wall's job is to stop the creature
seeing the target at all, which is a different question asked elsewhere.

The correction only ever *shortens* the answer, never lengthens it, because the ray is capped
at the same range used to decide whether to cast it at all.

## `report_swing`

**Contract** — record the outcome of one swing attempt and, if the recent history is
unanimous, move the inner edge one step. Called by the creature's attack state after each
swing resolves. Cheap; no allocation.

```text
FUNCTION report_swing(connected : bool)
  push connected onto the front of swing_history, dropping the oldest

  IF NOT every entry in swing_history equals connected
    RETURN                            # the run is broken; do not move the edge

  IF connected
    current_min = min(current_min + adapt_step, min_attack_distance)
  ELSE
    current_min = max(current_min - adapt_step, floor_distance)
```

**Notes** — the history is two entries long, so "unanimous" means *this swing and the previous
one agreed*. One is too jumpy and three would make the creature slow to respond within a single
bout; two is the smallest window that rejects a single fluke. That the length is a fixed
compile-time constant rather than a tuned number is incidental — a rebuild may make it
authored, at the cost of a per-creature array.

The adaptation is monotone in a run: five connecting swings move the edge five steps, snapping
at the ceiling. Nothing decays it back over time; the only reset is `begin_attack`.

## `can_start_melee`

**Contract** — yes when the creature can see the target *right now* (not from memory) and the
measured reach is inside the inner edge. The sight test comes first and short-circuits the
ray.

```text
FUNCTION can_start_melee(target) -> bool
  IF NOT creature.sees_right_now(target) THEN RETURN false
  RETURN distance_to_enemy(target) < min_distance()
```

## `should_stop_melee`

**Contract** — yes when the measured reach has passed the outer edge. Deliberately *not* the
negation of `can_start_melee`: there is no sight test, so a creature that loses sight of its
target mid-bout keeps swinging until the target is physically out of reach, and the gap
between the inner and outer edges is hysteresis that stops a target hovering at the boundary
from making the creature stutter between swinging and closing.

```text
FUNCTION should_stop_melee(target) -> bool
  RETURN distance_to_enemy(target) > max_distance()
```
