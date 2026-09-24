# src/xrGame/ai/monsters/group_states/group_state_eat_drag_inline.h

> Grip a ragdoll by an authored set of bones and haul it backward into the innermost part of the
> creature's territory before eating.

**Needs** — [`group_state_eat_drag.h`](group_state_eat_drag.h.md) · [`../monster_home.h`](../monster_home.h.md) · [`../monster_cover_manager.h`](../monster_cover_manager.h.md) · [`../../CaptureBoneCallback.h`](../../../CaptureBoneCallback.h.md) · [Seam: Rigid-body dynamics](../../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics) · [Seam: Configuration (ltx)](../../../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence) · [Kinematics — chapter 6](../../../../xrCore/README.md)
**Used by** — [`group_state_eat_drag.h`](group_state_eat_drag.h.md)
**Tier floor** — T2: reads the model's embedded configuration, resolves bone identifiers, and hands
a predicate to the physics layer that it calls back during capture

## Purpose

The only state in this chapter that reaches all the way down to the rigid-body seam. Dragging is
what makes a pack's kills disappear into the undergrowth instead of piling up in the open, and
doing it convincingly requires gripping the *right part* of the body: a creature that grips a
ragdoll by the hand drags a body that flails, and one that grips by the torso drags a body that
follows.

Which bones are grippable is **authored in the model itself**, not in the creature's section: the
skinned-mesh format carries an embedded configuration block, and this state reads a named section
of it. That makes the grip a property of the thing being dragged, which is correct — a mutant and
a human corpse have different bones worth holding.

## State

```text
RECORD GroupDragState
  cover_position       : vector   # where we are hauling to
  cover_vertex         : int      # its navigation vertex, or absent
  failed               : bool     # the grip could not be taken at all
  corpse_start_position: vector   # the corpse's position when the grip was taken
```

The corpse's start position exists only for the fallback completion test, used when no destination
vertex could be found.

## `initialize`

**Contract** — read the grippable bone list from the corpse's model, resolve each name to a bone
identifier, attempt the physical capture with a predicate that accepts a bone if that bone or any
of its ancestors is in the list, then choose a destination. Fails cleanly — setting the failure
flag and returning — if the model declares no grippable bones or if the capture is refused.

```text
FUNCTION initialize()
  model_config = the corpse's model's embedded configuration
  IF it has no "capture_used_bones" / "bones" entry
    failed = true; RETURN

  grippable = resolve each named bone to an identifier

  # the physics layer offers candidate bones; accept one whose chain reaches a grippable bone
  accept(bone) = walk from bone up the skeleton to the root;
                 true if any ancestor (or the bone itself) is in grippable

  physics.capture(corpse, accept)

  IF the capture succeeded
    cover_vertex = home.a_place_in_the_inner_region()
    IF that exists
      cover_position = navigation.position_of(cover_vertex)
    ELSE
      cover_position = self.position

    # reject a destination that is too close, absent, or outside the inner region
    IF cover_vertex is absent
       OR distance(self, cover_position) < 2
       OR NOT home.at_inner_region(cover_position)
      point = cover_system.find_cover(around the home point,
                                      min_radius = 1, max_radius = home.inner_radius)
      IF point EXISTS
        cover_vertex   = point.vertex
        cover_position = navigation.position_of(cover_vertex)
  ELSE
    failed = true

  corpse_start_position = corpse.position
  path.prepare()
```

**Notes** — the *ancestor walk* in the acceptance predicate is the interesting decision. The model
authors name a handful of bones — typically the spine and pelvis — and the predicate accepts any
bone whose chain passes through one of them. That means a creature can grip a shoulder or a thigh
and still be holding the body rather than a limb, because those bones descend from the spine. A
rebuild that matches only the named bones exactly will refuse most captures the original accepts.

The destination selection has two tiers and both aim at the **innermost** ring of the home region,
not at cover per se. The first tier asks the home component for a place it already knows; the
second falls back to a cover search anchored at the home point and bounded by the inner radius. A
destination less than 2 units away is rejected as pointless — the creature would drag the body a
step and stop.

When no destination survives either tier, the state still runs: the execute body switches to
"retreat from the corpse's own position", and completion is measured by distance travelled rather
than by arrival. That is a genuine graceful degradation and worth preserving.

The bone identifier array is allocated on the call stack in the original, sized at run time. A
rebuild allocates it however it likes; the fact that survives is that the list is short and
per-corpse.

## `execute`

**Contract** — do nothing at all if the grip failed. Otherwise request the drag action with the
"moving backward" animation flag, hand the destination to the path builder (or a retreat-from
directive when there is no destination), apply the generic path parameters and the calm
acceleration profile.

**Notes** — the backward-movement flag is what makes the creature walk *backward* while dragging,
which is the only way a quadruped holding something in its jaws can move without dragging the body
through its own legs. It is an animation flag, not a movement mode; the path is still forward
along the route.

## `finalize` / `critical_finalize`

**Contract** — both release the physical grip if one is held. Identical bodies; the distinction
between a clean and a forced exit carries no difference here, which is correct — a gripped ragdoll
must be dropped either way.

**Invariants** — releasing is guarded on actually holding, so a double release is impossible. This
is the only resource this state acquires and it is released on both paths, which is exactly what
the feeding composite above it fails to do (see
[`group_state_eat_inline.h`](group_state_eat_inline.h.md)).

## `check_completion`

**Contract** — finished when the attempt failed, when the grip has been lost, when the creature has
reached within 2 units of its destination, or — in the no-destination case — when the creature has
travelled further from the corpse's original position than the home region's inner radius.

**Notes** — the grip can be lost without this state doing anything: the physics layer drops a
capture when the constraint is violated, which happens when the corpse snags on geometry. Treating
that as *completion* rather than as failure is right, because the body has been moved and the
creature can eat where it stands.
