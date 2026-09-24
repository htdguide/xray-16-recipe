# src/xrGame/ai/monsters/states/monster_state_eat_inline.h

> The feeding behaviour of a solitary creature: approach a corpse, sniff it, optionally drag it
> away, eat until no longer hungry, then retreat and rest — with the corpse claimed against the
> rest of the squad for the whole sequence.

**Needs** — [`monster_state_eat.h`](monster_state_eat.h.md) · [`state_data.h`](state_data.h.md) · [`state_move_to_point.h`](state_move_to_point.h.md) · [`state_hide_from_point.h`](state_hide_from_point.h.md) · [`state_custom_action.h`](state_custom_action.h.md) · [`monster_state_eat_eat.h`](monster_state_eat_eat.h.md) · [`monster_state_eat_drag.h`](monster_state_eat_drag.h.md) · [`../monster_corpse_manager.h`](../monster_corpse_manager.h.md) · [`../ai_monster_squad_manager.h`](../ai_monster_squad_manager.h.md) · [`../monster_home.h`](../monster_home.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`monster_state_eat.h`](monster_state_eat.h.md)
**Tier floor** — T2: asks the rigid-body layer for the position of the ragdoll part nearest to the
creature, so it must address a live physics body; everything else is plain decision logic

## Purpose

This is the composite behaviour selected when a creature's brain decides it is hungry and a corpse
is available. It is a *scripted sequence with two loops*, not a planner: which leaf state runs next
is decided purely from which one just finished, so the whole behaviour is a small transition
function over the previous leaf.

Two facts give the sequence its shape. First, the corpse is a **shared resource**: the squad
manager holds a claim on it for exactly as long as this behaviour runs, so two pack members never
converge on the same body. Second, hunger is a *timer on this state object*, not the creature's
satiety value — the behaviour refuses to restart for twenty seconds after the last bite regardless
of how hungry the creature actually is.

A parallel implementation exists for pack animals in
[`../group_states/group_state_eat_inline.h`](../group_states/group_state_eat_inline.h.md); this one
is the solitary version, used by the creatures whose state manager registers it directly.

## State

```text
RECORD EatState
  corpse         : optional<entity>   # snapshot taken on entry; identity, not a live query
  time_last_eat  : int (milliseconds) # 0 means "never ate in this object's lifetime"
```

The snapshot is the load-bearing field. The corpse the behaviour *started* on is compared against
the corpse the memory component currently offers on every update; a divergence ends the behaviour
immediately. Without that snapshot a creature would silently re-target mid-approach and the squad
claim would be left on the wrong body.

**Invariant** — the squad claim on `corpse` is held from entry to exit on *both* exit paths, clean
and forced. This is the only resource the behaviour acquires.

**Invariant** — `corpse` is cleared when the engine announces that entity's destruction, and every
subsequent test compares against the memory component's answer rather than dereferencing the
snapshot.

## `check_start_conditions`

**Contract** — the behaviour may begin only when all four hold: the memory component names a
corpse; that corpse lies inside the creature's home region; the creature is hungry by this
object's own timer; and no squad member has already claimed that corpse.

**Notes** — the home-region test is what keeps a pack from streaming across a level toward a body
the player dropped somewhere else. It tests the *corpse's* position, not the creature's.

## `check_completion`

**Contract** — finished when the memory component now names a different corpse than the one
entered on, or when the hunger timer says the creature is no longer hungry.

**Notes** — note the asymmetry with the start condition: starting also requires the corpse to be
at home and unclaimed, but finishing does not re-test either. A corpse that is dragged out of the
home region mid-meal does not interrupt the meal, which is deliberate — dragging is one of the
steps.

## `hungry`

**Contract** — true when nothing has been eaten yet in this object's lifetime, or when twenty
seconds have elapsed since the last bite finished.

```text
FUNCTION hungry() -> bool
  RETURN time_last_eat == 0 OR time_last_eat + 20000 < now()
```

**Notes** — twenty seconds is hard-coded, not authored. The creature's satiety is changed by the
eating leaf and is what the rest of the game reads; this timer exists only to stop the behaviour
retriggering the instant it exits, which would pin a creature to a corpse forever. The number is
not derivable from anything in the source.

## `reselect_state` — the sequence

**Contract** — choose the next leaf from the one that just completed. Pure function of the previous
leaf plus two predicates (can this creature drag, and is the creature close enough to bite).

```text
FUNCTION next_leaf(previous) -> leaf
  IF previous is none            RETURN approach_at_a_run
  IF previous == approach_at_a_run   RETURN sniff
  IF previous == sniff
    IF creature.can_drag         RETURN drag
    RETURN eat_if_close_enough_else_approach_at_a_walk
  IF previous == drag            RETURN eat_if_close_enough_else_approach_at_a_walk
  IF previous == eat
    time_last_eat = now()
    IF NOT hungry()              RETURN retreat
    RETURN approach_at_a_walk
  IF previous == approach_at_a_walk
                                 RETURN eat_if_close_enough_else_approach_at_a_walk
  IF previous == retreat         RETURN rest
  IF previous == rest            RETURN rest        # terminal: loops until completion fires
```

**Notes** — three structural decisions live in this table.

*The approach appears twice, once running and once walking.* The first approach is the long one
from wherever the creature heard about the corpse; every later approach is a short correction after
a bite ended because the body shifted. Running the correction would look frantic, so it walks.

*"Eat if close enough, else walk" is a retry loop with no counter.* The eating leaf's own
start-condition test is the distance test, and failing it sends the creature back to walking. Two
leaves therefore ping-pong until either the creature gets close or the outer completion test fires.
There is no bound on that loop other than the hunger timer, which only advances when a bite
actually happens — so a creature that can never reach the body walks toward it until something else
interrupts. That is a real behaviour a rebuild reproduces, not a defect to fix quietly.

*Rest is terminal and self-looping.* It is not an exit: the sequence sits in rest, re-entering it
each time its timeout expires, until the outer `check_completion` notices hunger has ended or the
corpse has changed. Resting after a meal is therefore the default resting place of a fed creature.

## `setup_substates` — the authored parameters of each leaf

**Contract** — when a leaf is entered, fill its parameter record. Each leaf is a generic
movement/action state (see [`state_move_to_point.h`](state_move_to_point.h.md),
[`state_hide_from_point.h`](state_hide_from_point.h.md),
[`state_custom_action.h`](state_custom_action.h.md)); this is the only place the numbers are set.

```text
approach_at_a_run:
  point            = nearest_reachable_part_of(corpse)
  vertex           = unknown, let the path builder resolve it
  gait             = run, accelerating, braking on arrival, calm acceleration profile
  completion_dist  = section key "distance_to_corpse"
  ambient sound    = idle voice, delay = section key "idle_sound_delay"

sniff:
  action   = stand idle
  time_out = 1500 ms                       # hard-coded
  sound    = eating voice, delay = section key "eat_sound_delay"

retreat:
  flee from       = the corpse's position
  gait            = walk forward, accelerating, braking, calm profile
  distance        = 15                     # hard-coded: how far counts as away
  cover search    = prefer cover 20..30 away, searched within 25 of here
  ambient sound   = idle voice

rest:
  action   = rest
  time_out = 8500 ms                       # hard-coded
  sound    = idle voice

approach_at_a_walk:
  as approach_at_a_run, but walking
```

**Notes** — the split between authored and hard-coded is sharp and worth stating plainly. Only
three numbers here come from the creature's configuration section: how close counts as "at the
corpse" (`distance_to_corpse`) and the two ambient-voice repeat delays (`idle_sound_delay`,
`eat_sound_delay`). Every timeout and every distance in the retreat — 1500, 8500, 15, 20, 30, 25 —
is compiled in and identical for every creature in the game.

*Why the nearest ragdoll part, and not the corpse's origin.* A corpse with an active physics body
has drifted from the position its entity record reports: the pelvis may be metres from where the
entity "is". Pathing to the entity origin therefore stops the creature short of, or inside, the
body. The state asks the physics layer for the position of the ragdoll element nearest to the
creature and paths to that. A corpse with no active body, or one whose simulation has been put to
sleep, falls back to the entity position, which is then correct. Both approach leaves do this
independently and identically.

*The retreat's cover parameters describe a preference, not a requirement.* The retreat leaf looks
for a covered spot in the 20-to-30 band while fleeing; if it finds none it simply runs 15 units
from the corpse. The three cover numbers and the flee distance are independent and inconsistent
with each other on purpose — the creature stops at 15 whether or not it reached the cover band.

## `initialize` / `finalize` / `critical_finalize`

**Contract** — entry snapshots the corpse and claims it with the squad. Both exits release the
claim. Nothing else.

**Notes** — the two exits are byte-identical, which is correct here: a claim must be dropped
whether the behaviour ended cleanly or was pre-empted by a higher-priority one. The release asks
the memory component for the corpse again rather than using the snapshot, so if the memory
component has re-targeted between entry and exit, **the claim on the original body is leaked and a
claim on a body that was never claimed is released**. The squad's claim table is a plain list of
claimed bodies with no reference counting and a remove-one release, so the leak is silent and lasts
until the squad is torn down. A rebuild should release the *snapshot*, which is what this file already stores for
exactly this kind of reason.
