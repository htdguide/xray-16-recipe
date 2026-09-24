# src/xrGame/ai/monsters/states/monster_state_eat_drag_inline.h

> Grip a corpse, walk backward dragging it to a covered spot within thirty units, and let go.

**Needs** — [`monster_state_eat_drag.h`](monster_state_eat_drag.h.md) · [`../monster_cover_manager.h`](../monster_cover_manager.h.md) · [`../monster_corpse_manager.h`](../monster_corpse_manager.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — [`monster_state_eat_drag.h`](monster_state_eat_drag.h.md)
**Tier floor** — T2: acquires and releases a constraint in the rigid-body layer and must observe
that layer dropping it unilaterally

## Purpose

The one leaf of the solitary feeding sequence that reaches through to the physics seam. Its point
is world-visible: a creature that drags its kill into the bushes leaves the open ground clear,
which is what makes a pack's territory read as inhabited rather than littered.

Unlike the pack version in
[`../group_states/group_state_eat_drag_inline.h`](../group_states/group_state_eat_drag_inline.h.md),
this one does not choose which bones may be gripped — it asks the physics layer for a default
capture and accepts whatever it gets — and it aims at *cover*, not at the home region's interior.

## State

```text
RECORD DragState
  cover_position        : vector
  cover_vertex          : optional<int>   # absent means "no destination was found"
  failed                : bool            # the grip was refused
  corpse_start_position : vector          # only used when there is no destination
```

**Invariant** — `failed` and a held grip are mutually exclusive, and every method short-circuits on
`failed` before touching the physics layer.

## `initialize`

**Contract** — request a physical grip on the corpse the memory component currently names. If the
grip is refused, set the failure flag and do nothing else. If it is taken, look for cover between
ten and thirty units away from the creature's own position and record it as the destination; if no
cover is found, leave the destination absent. Snapshot the corpse's position either way, then
prepare the path builder.

```text
FUNCTION initialize()
  physics.capture(corpse)
  IF physics.capture_held AND NOT physics.capture_failed
    point = cover_system.find_cover(from = self.position, min = 10, max = 30)
    IF point EXISTS
      cover_position = point.position
      cover_vertex   = point.vertex
    ELSE
      cover_vertex   = absent
  ELSE
    failed = true
  corpse_start_position = corpse.position
  path.prepare()
```

**Notes** — the cover search is anchored at the *creature*, not at the corpse and not at the home
point. That is the difference that makes this the solitary behaviour: a lone animal drags the body
to the nearest concealment it can see from where it is standing, whereas a pack hauls kills back
to a shared place.

Ten and thirty are hard-coded and shared with several other states in this directory as a generic
"near cover" band; nothing derives them.

## `execute`

**Contract** — do nothing if the grip failed. Otherwise request the drag action with the
"moving backward" animation flag, hand the path builder the destination — or, when there is none, a
directive to retreat from the corpse's *current* position — apply the generic path parameters and
the calm acceleration profile.

**Notes** — the backward-movement flag is an animation selector, not a movement mode: the route is
still walked forward along its own direction. A quadruped with something in its jaws cannot move
forward without walking into the body, so the animation plays the reverse gait.

The no-destination fallback retreats from the *corpse's live position*, which is being dragged and
therefore moves with the creature. The retreat direction is consequently stable — the body trails
behind — and the creature walks away in a roughly straight line rather than circling. This reads
like an accident and works like a design.

## `finalize` / `critical_finalize`

**Contract** — both release the grip if one is held. Identical bodies; a gripped body must be
dropped whether the leaf ended or was pre-empted.

**Invariants** — the release is guarded on actually holding, so a double release cannot happen.

## `check_completion`

**Contract** — finished on any of: the grip was never taken; the grip is no longer held; the
destination exists and the creature is within two units of it; or the destination does not exist
and the creature has moved more than twenty units from where the corpse started.

**Notes** — losing the grip counts as *success*, not failure. The physics layer drops a capture
when its constraint is violated, which happens when the body snags on geometry — and at that point
the body has in fact been moved, so eating where the creature stands is the right next step.

Two and twenty are hard-coded. Two is an arrival tolerance around a navigation vertex; twenty is
"far enough to count as dragged" when no destination was ever chosen. Nothing derives either, and
the two paths are not calibrated against each other: a creature that found cover ten units away
stops after ten, one that found none drags for twenty.
