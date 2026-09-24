# src/xrGame/ai/monsters/control_animation_base_update.cpp

> The per-frame loop that picks the creature's clip from its action and its path, then reconciles the clip's speed, the body's speed and the turn rate.

**Needs** — [`control_animation_base.h`](control_animation_base.h.md) · [`control_direction_base.h`](control_direction_base.h.md) · [`control_movement_base.h`](control_movement_base.h.md) · [`control_path_builder_base.h`](control_path_builder_base.h.md) · [`detail_path_manager.h`](../../detail_path_manager.h.md) · [`monster_velocity_space.h`](monster_velocity_space.h.md) · [`basemonster/base_monster.h`](basemonster/base_monster.h.md) · [Seam: Rigid-body dynamics](../../../../SYSTEM-REQUIREMENTS.md#seam-rigid-body-dynamics)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2: runs every frame for every live creature and reads back the physics body's velocity

## Purpose

The heart of a creature's animation. Once per frame it answers two questions in order:
*which clip*, and *how fast* — where "how fast" means three numbers that must agree, the
clip's playback rate, the body's linear speed and the body's turn rate. Getting them to
agree is the entire difficulty: the path builder wants a speed, the clip was authored at a
speed, and the physics produces a third speed that is neither.

It also raises the *velocity bounce* event, which is the chapter's collision detector for
airborne abilities.

## State

Reads and writes the animation base's fields; see
[`control_animation_base.h`](control_animation_base.h.md).

## `update_frame`

**Contract** — the frame entry point. Runs the selection pass, then samples the body's
speed and raises the bounce event if it changed sharply. Does nothing while an attack clip
owns the body — an attack is allowed to finish without the selector second-guessing it.

```text
FUNCTION update_frame()
  IF NOT state_attack THEN
    IF moving on a path AND the path is enabled THEN
      face along the path        # reversed when the "move backwards" intent bit is set
    select_animation()
    select_velocities()
    IF the chosen motion differs from the one playing THEN
      start it
  check_velocity_bounce()
```

## `SelectAnimation`

**Contract** — choose the motion for this frame and write it into the current-animation
record. Consults, in order: a script override (which wins outright), the action derived
from the path when the creature is moving, otherwise the action the brain set; then the
creature's own intent bits; then a posture transition; then the conditional substitutions
and the turn clips.

```text
FUNCTION select_animation()
  IF an override motion is set THEN
    set current motion <- override; RETURN

  action <- IF moving on an enabled path THEN action_from_path() ELSE brain's action
  set current motion <- motions[action].anim

  ask the creature to refresh its intent bits

  IF the motion changed AND a transition exists from the old to the new THEN
    play the bridge clip instead; RETURN    # the transition owns this frame

  apply conditional substitutions
  apply turn clips
```

**Notes** — the ordering is the decision. A transition short-circuits everything after it,
so a creature standing up does not also get its wounded substitution or its turn clip for
the duration of the bridge. And the path overrides the brain: a brain that says "stand
idle" while the path builder is moving the creature gets the path's action anyway, which
is what keeps a creature from sliding along the ground in an idle pose.

## `SetTurnAnimation`

**Contract** — replace the chosen clip with a left- or right-turn variant when the creature
is facing the wrong way. Two conditions admit a turn clip: the creature is in a standing
idle clip in the standing posture with any heading error at all, or the creature is moving
along a path with a heading error above thirty degrees.

**Notes** — the thirty degrees is authored nowhere; it is a constant in this file. Below
it, a moving creature simply turns while walking; above it, it plays a dedicated turning
clip. The asymmetry with the standing case — where *any* error triggers a turn clip — is
what makes idle creatures look alert and moving creatures look smooth.

## `SelectVelocities`

**Contract** — reconcile the three speeds. Reads the speed the path's current waypoint
calls for, the speed the chosen clip was authored at, and the speed the physics body is
actually achieving; writes a linear speed to the movement channel, a playback rate to the
animation, and a turn rate to the direction channel. Does not allocate. This is the single
densest decision in the chapter, so it is given as sub-steps.

```text
FUNCTION select_velocities()
  # 1. what the path wants
  path_speed <- zero
  IF moving on a path THEN
    v <- speed index at the current waypoint
    # a creature standing at a waypoint with another waypoint ahead is mid-turn:
    # once the turn finishes, adopt the next waypoint's speed rather than staying stopped
    IF v IS "stand" AND a next waypoint exists AND NOT turning THEN
      v <- speed index at the next waypoint
    path_speed <- the travel parameters for v

  # 2. what the clip was authored at
  anim_speed <- chosen clip's authored linear and angular speed

  # 3. drive the body
  IF the creature is invisible THEN
    body speed <- path_speed.linear        # no clip to match, so obey the path
  ELSE IF anim_speed.linear IS zero THEN
    stop, bounded by the braking rate
  ELSE IF NOT check_braking(-2, anim_speed.linear) THEN
    body speed <- anim_speed.linear        # the clip leads, the body follows
  ELSE
    decelerate                             # braking owns the body this frame

  # 4. match the clip's rate to the ground
  IF visible AND anim_speed.linear IS NOT zero THEN
    IF chain_lookup(measured body speed, chosen motion) succeeds THEN
      adopt the chain's motion and its playback rate
    ELSE
      playback rate <- "authored"          # a negative sentinel meaning "as authored"
  ELSE
    playback rate <- "authored"

  # 5. turn rate
  IF the creature is invisible THEN
    turn rate <- path_speed.angular
  ELSE IF the brain's action is attack THEN
    turn rate <- clip's angular speed * the creature's melee rotation factor
  ELSE
    turn rate <- clip's angular speed
```

**Notes** — step 3 is the chapter's central convention, and it inverts the obvious one:
**the animation drives the body, not the other way round.** A visible creature moves at
the speed its clip was authored at, and the path builder's waypoint speeds only ever
select *which* clip. The invisible case — a cloaked creature with nothing to look at —
flips it back and obeys the path directly, which is why an invisible creature can move at
speeds no clip covers.

Step 4 then closes the loop: the physics body does not exactly achieve the commanded speed
(slopes, friction, collisions), so the chain lookup re-picks a clip from the *measured*
speed and stretches its playback rate to remove the residual slip. A rebuild that skips
this gets feet that slide.

The "authored" sentinel is a negative playback rate. It is a sentinel, not a rate, and a
rebuild should give it a name.

The melee rotation factor is a per-creature authored number applied only during attack
clips, and the source flags it as something that should have been a general external
factor rather than a special case.

Two commented-out assertions in the original demanded that the path's speed and the clip's
speed agree. They do not agree in the shipped data, which is precisely why step 4 exists.

## `CheckVelocityBounce`

**Contract** — sample the physical body's speed and raise the bounce event when the ratio
between this frame's and last frame's speed exceeds one and a half in either direction.
The event carries the ratio, negated when the creature slowed. Zero speeds are floored at
one hundredth before the ratio is taken, so a creature starting from rest or coming to a
complete stop still produces a finite, large ratio and the event fires.

**Notes** — this is how a creature in mid-air learns it has hit something, and it is the
only such signal: a jump subscribes to the event and treats a large *deceleration* as a
landing or an impact. Using a ratio rather than a difference makes the threshold
speed-independent, which matters because the chapter's creatures range from a rat to a
pseudogiant. The threshold of one and a half is a constant in this file with no recorded
derivation.

## `set_override_animation` / `clear_override_animation`

**Contract** — force one clip and suppress the selector, or stop doing so. Setting the same
override twice is a no-op; setting a different override while one already stands is an
error. The by-name spelling resolves a full clip name — prefix plus decimal index — by
scanning the motion table for a registered prefix the name starts with, then parsing the
remainder as the variant index; a name matching no registered prefix is a hard failure.

**Notes** — this is the script layer's way to puppet a creature, and the prefix-scan is the
only place the clip-naming convention is *parsed* rather than generated. It is
first-match, so a creature that registers both `stand_` and `stand_idle_` resolves
`stand_idle_2` against whichever the table iteration reaches first. No shipped creature
has overlapping prefixes.

## `ScheduledInit`

**Contract** — clears the intent bits and disables acceleration. Runs once per spawn, after
the creature's own load has built the tables.
