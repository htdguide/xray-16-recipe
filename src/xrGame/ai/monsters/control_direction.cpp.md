# src/xrGame/ai/monsters/control_direction.cpp

> The direction resource: it eases the creature's heading and pitch toward their targets each frame, writes the result into the model's transform, and reports when a rotation completes.

**Needs** — [`control_direction.h`](control_direction.h.md) · [`control_manager.h`](control_manager.h.md) · [`control_path_builder.h`](control_path_builder.h.md) · [`detail_path_manager.h`](../../detail_path_manager.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [`xrAICore/Navigation/level_graph.h`](../../../xrAICore/Navigation/level_graph.h.md) · [`xrAICore/Navigation/ai_object_location.h`](../../../xrAICore/Navigation/ai_object_location.h.md)
**Used by** — reached through its declarations in [`control_direction.h`](control_direction.h.md); callers name that, not this file.
**Tier floor** — T2: per-frame angle integration plus a transform write for every live creature

## Purpose

One of the four body resources. Whoever holds it writes a target heading, a target pitch
and the rates to approach them into the channel's payload; this element integrates toward
those targets each frame and stamps the result onto the creature's model transform. It
also owns the *pitch correction* that makes a creature lie along the slope it is standing
on rather than floating level.

Its services half — "am I facing that", "is it to my left or right", "how far must I turn"
— is used by nearly every ability in the chapter, so the element is as much a query
surface as a driver.

## State

```text
RECORD DirectionResource
  heading : { current_angle, current_speed, current_acceleration : real }
  pitch   : { current_angle, current_speed, current_acceleration : real }

RECORD DirectionPayload            # what a capturer writes
  heading : { target_angle, target_speed : real }
  pitch   : { target_angle, target_speed : real }
  linear_dependency : bool         # scale the turn rate by how fast the body is moving
```

**Invariants** — both current angles are kept normalized; heading to a full turn, pitch to
a signed half turn either side of level. Accelerations start unbounded, meaning the speed
reaches its target in one frame unless a capturer says otherwise.

The heading is the authority on where the creature's body points: the path builder's body
orientation is *written from* here every frame, not read into it.

## `update_frame`

**Contract** — integrate both angles one frame, write them through to the model transform,
and raise the rotation-end event for whichever axis just arrived. Runs every frame while
the channel is active.

```text
FUNCTION update_frame()
  pitch_correction()                       # may rewrite the pitch target

  # pitch rate is derived, not commanded: proportional to the remaining error,
  # four times over, clamped to between a twelfth and five twelfths of a turn per second
  diff <- angle_difference(pitch.current, payload.pitch.target) * 4
  clamp diff INTO [PI/6, 5*PI/6]
  payload.pitch.target_speed <- diff
  pitch.current_speed        <- diff

  # heading rate: when the body is moving and linear dependency is on, the turn rate is
  # scaled by how much of the target speed the body has actually reached, so a creature
  # that has not got up to speed does not pivot faster than it walks
  IF body is moving AND target speed non-zero AND linear_dependency THEN
    heading.current_speed <- payload.heading.target_speed
                             * current body speed / target body speed
  ELSE
    ease heading.current_speed toward payload.heading.target_speed at its acceleration

  arrived_heading <- angles were apart, now equal
  ease heading.current_angle toward payload.heading.target_angle at heading.current_speed

  ease pitch.current_speed toward payload.pitch.target_speed at its acceleration
  arrived_pitch <- angles were apart, now equal
  ease pitch.current_angle toward payload.pitch.target_angle at pitch.current_speed

  # publish: the path builder's body orientation is this element's output
  path_builder.body.speed          <- heading.current_speed
  path_builder.body.current.yaw    <- heading.current_angle
  path_builder.body.target.yaw     <- heading.current_angle
  path_builder.body.current.pitch  <- pitch.current_angle
  path_builder.body.target.pitch   <- pitch.current_angle

  IF the model is not being moved by an animation THEN
    set the model transform's orientation from the two angles, negated
  restore the model's position, which the orientation write clobbered

  IF either axis arrived THEN raise rotation_end carrying which
```

**Notes** — three things here a rebuild must reproduce and would not guess.

The angles are **negated** when they reach the transform. The AI layer's heading grows the
opposite way from the renderer's; the sign flip appears at every boundary between them in
this chapter and is not localized anywhere.

Setting the transform's orientation destroys its translation, so the position is saved and
restored around the write. That is an artefact of the matrix helper, but the *requirement*
survives: orientation and position are set independently here.

The linear-dependency rule is the reason creatures do not spin on the spot while
accelerating. A capturer that wants an exact angular rate — a jump matching its turn to
its flight time, say — switches it off, and every such capturer is careful to switch it
back on when it releases.

The pitch rate is *never* commanded by a capturer: whatever a capturer writes into the
pitch target speed is overwritten at the top of this function by a value derived from the
remaining error. The clamp bounds it to between a twelfth and five twelfths of a turn per
second. Those two numbers and the factor of four are constants here with no recorded
derivation.

## `pitch_correction`

**Contract** — choose the creature's pitch target from the ground it is on, when the
creature's class enables the feature. Two sources, in order: the slope of the next path
segment when the creature is moving and the segment is longer than one world unit;
otherwise the plane of the navigation cell the creature stands on, projected along the
creature's own facing.

```text
FUNCTION pitch_correction()
  IF the creature does not support pitch correction THEN RETURN

  IF moving on a path AND a next waypoint exists THEN
    IF squared distance between this waypoint and the next > 1 THEN
      pitch target <- -pitch of (next waypoint - current waypoint)
      RETURN

  plane <- the plane of the navigation cell under the creature
  here  <- the creature's position projected onto that plane
  ahead <- (here + the creature's facing) projected onto that plane
  pitch target <- -pitch of (ahead - here)
```

**Notes** — the two sources answer different questions and the distinction matters. The
path-segment source is what lets a creature *climb a wall*: a path that leaves the floor
gives a steep segment and the creature pitches to match, which is the whole mechanism
behind the wall-crawling creatures. The cell-plane source handles standing on a slope.

The one-world-unit threshold exists because two waypoints closer than that give a
direction dominated by noise. It is a constant here.

The navigation cell's plane is stored compressed in the level data and is decompressed on
every call; that is a data-format concern, not a decision.

## `is_face_target`

**Contract** — is the given position, or the given object, within a heading error of the
creature's *facing direction*. Pitch is ignored. Used by every ability that requires the
creature to be looking at its enemy before it may start.

**Notes** — it compares against the creature's rendered facing, not against this element's
current heading angle, and those differ during a turn. Callers that want the internal
angle use `get_heading` instead. Nothing records whether the difference was intended.

## `is_from_right`

**Contract** — is the given position, or the given heading, to the right of the creature's
current heading. The sign that decides left-turn clip versus right-turn clip, and which
side a rotation jump goes.

## `is_turning`

**Contract** — are the current and target headings apart by more than a tolerance. The
tolerance defaults to the smallest meaningful angle, so with no argument this answers "is
any rotation outstanding at all".

## `get_heading` / `get_heading_current`

**Contract** — read out the current and target heading angles. The pair form is how every
caller computes "how far must I still turn", by taking the difference itself.

## `angle_to_target`

**Contract** — the normalized heading that points at a world position, in this element's
sign convention — that is, negated relative to the renderer's. Every capturer that wants
to face something writes this into the payload's heading target.

## `reinit`

**Contract** — seed both current angles from the path builder's body orientation, zero both
speeds, set both accelerations unbounded, copy the body's target angles into the payload,
and enable linear dependency. The element comes up active, as every pure element does.
