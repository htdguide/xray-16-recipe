# src/xrGame/smart_cover_evaluators.cpp

> The bodies of the smart-cover world-state questions, including the two dwell timers that flip the creature between idling and looking out.

**Needs** — [`smart_cover_evaluators.h`](smart_cover_evaluators.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) · [`smart_cover_transition.hpp`](smart_cover_transition.hpp.md) · [`smart_cover_transition_animation.hpp`](smart_cover_transition_animation.hpp.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`enemy_manager.h`](enemy_manager.h.md)
**Used by** — reached through its declarations in [`smart_cover_evaluators.h`](smart_cover_evaluators.h.md); callers name that, not this file.
**Tier floor** — T2: cheap state queries on the planning path

## Purpose

Each of these answers one question the planner asks. Most are a single comparison and need
no explanation beyond their contract. Three are not, and they are what this file is for:
the transition guard, and the pair of dwell timers that make a creature in cover
alternate between hiding and peeking.

## State

Stateless, except that two evaluators carry a mutable dwell interval and two write back
into the animation planner. That is a deliberate violation of the evaluator contract and
is discussed under the dwell timers.

## The current/target comparisons

**Contract** —

- **`in_cover_evaluator`** and **`cover_entered_evaluator`** — true when the creature's
  current movement parameters name a cover.
- **`cover_actual_evaluator`** — true when the current cover is the target cover. Asserts
  there *is* a current cover: the planner only reaches this question having established
  the creature is in one, so a miss is a broken plan rather than a false answer.
- **`loophole_actual_evaluator`** — true when both the cover and the loophole match their
  targets. Unlike the previous one it does not assert, because "not in a cover at all"
  legitimately answers false here.

**Invariants** — all three compare the *movement manager's* current and target parameters,
not the planner's own idea of where the creature is. The movement manager is the authority
on where a creature actually is inside a cover, and routing these through it is what keeps
the plan honest about progress it has not actually made.

**Notes** — `loophole_actual_evaluator` is constructed with a loophole value and a
reference to the animation planner, and uses neither. They are vestigial; a rebuild should
drop them.

## `loophole_exitable_evaluator`

**Contract** — true when the current loophole has an authored way out. False when the
creature is not at a loophole at all. The flag is derived at load from the description's
transition graph ([`smart_cover_description.cpp`](smart_cover_description.cpp.md)).

## `can_exit_loophole_with_animation` — the transition guard

**Contract** — true when the creature is *moving somewhere* (to a different cover, or to a
different loophole of the same cover) **and** the transition it is about to make has an
animation to play. False when the creature is already where it wants to be. Asserts a
current cover exists.

```text
FUNCTION can_exit_with_animation() -> bool
  REQUIRE current cover exists
  IF current cover IS NOT target cover
    RETURN the pending transition has an animation
  IF current loophole IS NOT target loophole
    RETURN the pending transition has an animation
  RETURN false                      # nowhere to go
```

**Invariants** — this is the guard that stops a creature attempting a move it has no clip
for. Smart-cover movement is entirely animation-driven: a creature does not walk from one
loophole to another, it *plays a clip* that leaves it there. A transition with no clip is
therefore not a slower move, it is an impossible one, and the planner must be told so
rather than committing to it and stalling.

**Notes** — the two branches compute the same thing and differ only in what they report in
the development log: cover-level or loophole-level movement, naming the two endpoints, with
"outside the world of covers" printed for a missing endpoint. A rebuild can collapse them
and keep the distinction only in the diagnostic.

## `is_action_available_evaluator`

**Contract** — true when the creature is at a loophole of a cover and that loophole offers
the named action. False, not a failure, when either is missing — this evaluator is asked
*before* the planner knows where the creature is.

## `loophole_hit_long_ago_evaluator`

**Contract** — true when the configured interval has passed since the creature was last hit
while in this cover. The time of the last hit is recorded by the animation planner.

**Invariants** — the comparison is a plain "recorded time plus interval is in the past",
over a monotonic global millisecond clock. It is not robust to that clock wrapping; the
wrap is far beyond any play session and the original does not guard it.

## `loophole_planner_const_evaluator`

**Contract** — returns a fixed answer set at construction. It exists so a planner can
declare a world-state property whose value never changes within that planner, without
inventing a real question. A rebuild with a richer world-state representation should pin
the property directly and delete this.

## `default_behaviour_evaluator`

**Contract** — true when the creature's movement manager is in either *default* or *combat*
cover behaviour. The two are separate modes and this question deliberately does not
distinguish them: it means "the cover is driving the creature", as opposed to a script
having taken over.

## `can_fire_at_enemy_evaluator`

**Contract** — whether the creature may shoot from where it stands.

```text
FUNCTION can_fire_at_enemy() -> bool
  IF NOT in default behaviour                RETURN true    # someone else decided
  IF the current loophole is a fire position RETURN true
  RETURN the enemy is inside the loophole's arc
```

**Invariants** — the first arm is a deliberate abdication: in combat or scripted behaviour,
the decision to fire has already been made elsewhere and this evaluator must not veto it.
The second arm says an authored firing loophole always permits fire regardless of the arc.
Only the default case falls back to the geometric test.

## The dwell timers — `idle_time_interval_passed_evaluator` and `lookout_time_interval_passed_evaluator`

**Contract** — together these two make a creature in a cover alternate between staying down
and looking out. They are mirror images: each is true while *its* phase is running and
false when its phase expires, and each expiry hands over to the other by stamping the
other's start time, resetting its own interval to the planner's default, and flipping the
shared phase flag.

```text
FUNCTION idle_interval_passed() -> bool
  IF NOT in the idle phase                            RETURN false
  IF now <= last_idle_time + interval
    keep the idle phase
    RETURN true
  # expiry: hand over to lookout
  last_lookout_time = now
  interval          = planner's default idle interval
  leave the idle phase
  RETURN false

FUNCTION lookout_interval_passed() -> bool      # the mirror
  IF in the idle phase                                RETURN false
  IF now <= last_lookout_time + interval
    stay out of the idle phase
    RETURN true
  last_idle_time = now
  interval       = planner's default lookout interval
  enter the idle phase
  RETURN false
```

**Invariants** —

- The phase flag is **shared**: one of the two evaluators is live at a time and the other
  short-circuits to false. Without the guard both would run their timers and the creature
  would be idling and looking out at once.
- Each expiry stamps the *other* phase's start time. This is what makes the alternation
  seamless — the new phase begins at the moment the old one ended, not at the next
  planning cycle.
- Each evaluator resets **its own** interval to the planner's default on expiry. The
  purpose is that the *first* interval of a phase can be overridden (by the planner, to
  stagger a squad, or to cut an idle short after being hit) and every subsequent one
  reverts to the authored default.

**Notes** — these two evaluators **mutate world state while being evaluated**, which
everything else in the planner assumes does not happen. The consequence a rebuild must
respect: the planner may query an evaluator several times within one planning cycle, or
not at all if the property is already known, and the timing of the phase flip therefore
depends on the planner's search order. The behaviour is reproducible only if the phase flip
stays inside the evaluation. Pulling the timer out into the planner's own update — which is
where it belongs — changes when creatures peek, so it is a behavioural change, not a
refactor.
