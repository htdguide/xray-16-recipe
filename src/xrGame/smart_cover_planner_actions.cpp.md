# src/xrGame/smart_cover_planner_actions.cpp

> The three operators that get a creature from one loophole to another, or out of the cover — and the rule that a smart-cover action's effect lands when its clip ends, not when it is chosen.

**Needs** — [`smart_cover_planner_actions.h`](smart_cover_planner_actions.h.md) · [`smart_cover.h`](smart_cover.h.md) · [`smart_cover_description.h`](smart_cover_description.h.md) · [`smart_cover_transition.hpp`](smart_cover_transition.hpp.md) · [`smart_cover_transition_animation.hpp`](smart_cover_transition_animation.hpp.md) · [`smart_cover_animation_planner.h`](smart_cover_animation_planner.h.md) · [`stalker_movement_manager_smart_cover.h`](stalker_movement_manager_smart_cover.h.md) · [`sight_manager.h`](sight_manager.h.md) · [`stalker_animation_manager.h`](stalker_animation_manager.h.md) · [`animation_movement_controller.h`](animation_movement_controller.h.md) · [`ai/stalker/ai_stalker.h`](ai/stalker/ai_stalker.h.md) · [`stalker_decision_space.h`](stalker_decision_space.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: operator lifecycle around an animated move

## Purpose

Three operators, one idea: **movement inside a smart cover is a clip, and the creature's
position changes when the clip ends**. Committing the move at the start would put the
creature at the destination while the animation still shows it at the source; committing
it per-frame would fight the animation's own root motion. So each of these actions arms a
transition, lets the clip play, and commits on the end callback.

The second idea, threaded through all three, is that the creature's **aiming must be
switched off for the duration of an animated move**. The clip owns the whole body,
including the head; leaving the aiming manager running would have it fighting the clip for
the neck and shoulders.

## `action_base`

**Contract** — supplies the defaults: the two marker hooks do nothing, and an action is
animated unless it says otherwise.

### `setup_orientation`

**Contract** — re-enables the creature's aiming and reinstates its bone callbacks. Called
by every action at the moment it hands the creature back to ordinary control.

**Invariants** — the two must happen together. Re-enabling aiming resynchronizes the
creature's angles with the pose the clip left it in
([`sight_manager.cpp`](sight_manager.cpp.md)); reinstating the bone callbacks restores the
game-facing animation events the clip's playback suppressed. A rebuild that does one
without the other leaves either a snapping head or a creature whose footsteps are silent.

## `change_loophole` — the animated move

**Contract** — disables aiming on entry, plays the transition's clip, and on the clip's end
commits the creature to the next loophole. Re-enables aiming on finalization. The same
action type serves both moving within a cover and leaving it with an animation.

```text
FUNCTION initialize()     disable aiming
FUNCTION finalize()       enable aiming
FUNCTION on_animation_end()  movement.go_next_loophole()
```

### `select_animation`

**Contract** — names the clip. Two cases, distinguished by whether this is an *exit*
transition: an ordinary move plays the transition's own clip, while an exit plays the clip
of the transition that ends in the **target body state** — standing, crouching or lying —
because how a creature leaves a cover determines what posture it is in afterwards.

```text
FUNCTION select_animation() -> text
  IF NOT an exit transition
    RETURN the current transition's animation id
  animation = the current transition's animation for the target body state
  REQUIRE the description has an edge from the current loophole to the exit vertex
  REQUIRE that animation exists
  RETURN its animation id
```

**Invariants** — the two assertions guard authored data, not code: an exit the planner
chose but the description cannot express, or an exit whose clip is missing for the wanted
posture. The messages name the cover, the loophole and the exit endpoint, because that is
the triple an author needs to find the hole.

**Notes** — posture selection applies only on exit. A move between loopholes keeps
whatever posture the transition's single clip implies, because the destination loophole
fixes the posture anyway.

## `non_animated_change_loophole` — the walked move

**Contract** — reports itself not animated, so the creature keeps its ordinary locomotion.
On entry it disables and immediately re-enables aiming, switches the creature to running,
and tells the movement manager a non-animated loophole change is under way; on finalization
it tells it the change is over.

```text
FUNCTION initialize()
  disable aiming                       # forces a re-sync on the next enable
  setup_orientation()                  # enables aiming and restores bone callbacks
  movement type = run
  movement.start_non_animated_loophole_change()

FUNCTION finalize()
  movement.stop_non_animated_loophole_change()
  then the base finalization
```

**Invariants** —

- The disable-then-enable is not redundant. Enabling aiming is what re-derives the
  creature's angles from its actual world transform, and that only happens on a
  *transition* from disabled to enabled. Disabling first is how this action forces the
  resynchronization; the original says so in as many words.
- The movement manager must be told the change has *stopped* before the base finalization
  runs, because the base may hand the creature to another action that immediately starts
  its own move.
- Running, not walking, is not a style choice: the unanimated change is the fallback used
  when there is no clip, and a creature ambling between loopholes under fire reads as
  broken.

## `exit` — leaving, animated or not

**Contract** — the only action whose animated-ness is decided at run time, by asking
whether the pending transition has a clip. The two paths differ in *when* the creature is
committed to leaving.

```text
FUNCTION is_animated_action() -> bool
  RETURN the current transition has an animation

FUNCTION initialize()
  IF the transition has an animation THEN disable aiming
  # otherwise leave aiming alone; there is no clip to fight

FUNCTION execute()
  IF the transition has an animation THEN RETURN      # wait for the clip to end
  setup_orientation()
  movement.go_next_loophole()
  movement type = run

FUNCTION on_animation_end()
  setup_orientation()
  movement.go_next_loophole()
  movement type = run
```

**Invariants** — the commit is identical in both paths; only its trigger differs. With a
clip it is the clip's end; without one it is the action's first execution. Duplicating the
three steps rather than sharing them is incidental, but the *ordering* within them is not:
orientation is restored before the creature is committed, so the resynchronization reads
the pose the creature is actually in rather than the one it is about to be in.

**Notes** — `execute` runs every scheduled cycle, so the unanimated path commits and then
keeps re-committing until the plan moves on. That is harmless because committing to the
next loophole is idempotent once the creature has already arrived, and it is why no guard
exists. A rebuild should still guard it.
