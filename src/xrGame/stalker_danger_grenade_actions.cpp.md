# src/xrGame/stalker_danger_grenade_actions.cpp

> Surviving a grenade: the decision to throw the rifle over your shoulder and run, and everything that follows the blast.

**Needs** — [`stalker_danger_grenade_actions.h`](stalker_danger_grenade_actions.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`object_handler.h`](object_handler.h.md) · [`memory_manager.h`](memory_manager.h.md) · [`danger_manager.h`](danger_manager.h.md) · [`agent_manager.h`](agent_manager.h.md) · [`agent_member_manager.h`](agent_member_manager.h.md) · [`cover_point.h`](cover_point.h.md) · [`Inventory.h`](Inventory.h.md) · [`Weapon.h`](Weapon.h.md)
**Used by** — [`stalker_danger_grenade_actions.h`](stalker_danger_grenade_actions.h.md)
**Tier floor** — T2: per-cycle movement, sight and weapon configuration.

## Purpose

A live grenade is the one threat with a **deadline**, and that is what makes this branch
different from the other danger reactions. Everything before the blast is about covering
distance in the seconds available; everything after is the ordinary take-cover-and-sweep
shape. The single most characterful decision in the stalker AI lives in the first action:
a creature that is about to be caught by a grenade slings its weapon and sprints, and a
creature that has already slung it panics outright.

## State

Only the after-explosion reposition holds anything: the same coin-flipped sight preference
described in
[`stalker_danger_unknown_actions.cpp`](stalker_danger_unknown_actions.cpp.md).

## `DangerGrenadeTakeCover` — entry

**Contract** — clear both progress propositions and configure a run along the navigation
mesh. Notably it does **not** set a mental state: that is decided per cycle, because it
depends on what the creature manages to do with its weapon.

## `DangerGrenadeTakeCover` — per cycle

**Contract** — steer to the claimed cover point, and simultaneously decide what to do with
the weapon and how frightened to look. Returns early when the danger has expired.

```text
FUNCTION execute()
  base.execute()
  IF no danger selected THEN RETURN

  point := my claimed cover point
  IF point exists THEN steer to point ELSE head for nearest accessible position

  # weapon and composure, decided together
  IF nothing in hand THEN
    weapon_goal(IDLE);   mood := panic
  ELSE IF the held item is my best weapon AND it can be slung THEN
    weapon_goal(SLING, held item)
    mood := IF it is already slung THEN panic ELSE danger
  ELSE
    weapon_goal(IDLE);   mood := danger

  IF path not yet completed THEN
    movement.body_state := standing
    IF distance_to_destination > 2 THEN
      movement.mental_state := mood
      sight := watch along the path
    ELSE
      movement.mental_state := danger
      sight := watch the cover
    RETURN

  set property CoverReached = true
```

**Invariants** — the run gait is set at entry and never revisited, so the creature runs for
the whole action whatever happens to its weapon.

**Notes** — the weapon logic is the decision worth carrying. A stalker running from a
grenade tries to *sling its rifle*, because the slung carry is what unlocks the fastest
run animation set. Until the sling completes the creature is merely alarmed; once the
weapon is on its back it switches to outright panic, which is a different and faster
movement style. So the sequence reads, on screen, as a soldier throwing his rifle over his
shoulder and then genuinely bolting — and it falls out of three conditions rather than any
scripted sequence.

Weapons that cannot be slung, and items that are not the creature's best weapon, get
neither: the creature keeps hold of whatever it has and runs merely alarmed. That prevents
a stalker discarding a quest item or a detector to escape a grenade.

The composure also drops back to plain alarm in the last two units before cover, together
with the sight switching to the cover itself. A creature does not arrive panicking; it
arrives composed and facing the right way.

## `DangerGrenadeWaitForExplosion` — entry and per cycle

**Contract** — crouch behind the cover, stand still, keep the best weapon idle in hand, and
watch. Per cycle, the eyes alternate between the cover and over the top of it depending on
whether the body has finished turning — the same body-driven sequencing as the
unknown-danger sweep.

**Invariants** — this action has no completion condition of its own. It ends when the
grenade explodes, which the planner notices through an evaluator watching whether the
grenade object still exists. An action that waited on a timer would either stand the
creature up early or keep it down after a dud.

**Notes** — the weapon is *idled in hand*, not slung. The run is over; what matters now is
being able to shoot the moment the blast passes, since a grenade usually arrives with
someone behind it.

## `DangerGrenadeTakeCoverAfterExplosion`

**Contract** — identical in shape to
[`DangerUnknownTakeCover`](stalker_danger_unknown_actions.cpp.md): clear the progress
propositions, coin-flip the sight preference, run to the claimed cover with the weapon
aimed and ready, and report arrival.

**Notes** — the repeat is deliberate rather than duplicated by accident. The cover chosen
*before* the blast was chosen against the grenade's position; after the blast the threat
that matters is whoever threw it, and the evaluator will have picked a different point. So
the creature moves twice, and the second move is the one that puts it somewhere useful for
the fight. A rebuild can share one implementation between the two branches; the distinction
that matters is which evaluator supplied the point, not which action ran.

## `DangerGrenadeLookAround`

**Contract** — the same fifteen-second crouched sweep as the unknown-danger version, with
one difference: the weapon is brought to aimed-and-ready rather than left idle, because a
grenade implies a thrower.

## `DangerGrenadeSearch`

**Contract** — the branch terminator. Stands the creature still in a legal position, eyes
where they already point, weapon idle, and does nothing per cycle.

**Invariants** — it declares `Danger = false` as its effect and never writes it. Unlike the
unknown-danger terminator, it also publishes nothing: it simply occupies the creature until
the grenade danger record ages out of the danger memory on its own. The branch ends because
the threat expires, not because the creature resolved it.

**Notes** — the asymmetry with the unknown-danger branch is meaningful. An unexplained noise
makes a place dangerous for two minutes; a grenade that has already gone off does not,
because the danger has passed and the thrower's position is a combat problem rather than a
pathing one.
