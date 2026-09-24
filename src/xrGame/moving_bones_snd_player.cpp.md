# src/xrGame/moving_bones_snd_player.cpp

> Plays a looping sound at a bone, with its pitch driven by how fast that bone is rotating, so machinery sounds like it is working.

**Needs** — [`moving_bones_snd_player.h`](moving_bones_snd_player.h.md) · [`GameObject.h`](GameObject.h.md) · [`Include/xrRender/Kinematics.h`](../Include/xrRender/Kinematics.h.md) · [`xrCore/Animation/Bone.hpp`](../xrCore/Animation/Bone.hpp.md) · [`xrPhysics/matrix_utils.h`](../xrPhysics/matrix_utils.h.md) · [Seam: Audio device](../../SYSTEM-REQUIREMENTS.md#seam-audio-device)
**Used by** — reached through its declarations in [`moving_bones_snd_player.h`](moving_bones_snd_player.h.md); callers name that, not this file.
**Tier floor** — T2: per-frame matrix differencing and a sound source

## Purpose

A small optional attachment for any animated object. It watches one bone, differentiates
its world transform to get an angular speed, smooths that heavily, and drives a looping
sound's pitch from the ratio of the smoothed speed to an authored reference speed. Silent
while the bone is nearly still.

It exists as its own file because it is opt-in per object *in data*: the factory returns
nothing unless the object's configuration declares the feature, so an object pays nothing
for not using it.

## State

```text
RECORD MovingBonesSoundPlayer
  bone            : int          # index into the skeleton; the sentinel means unset
  min_factor      : real         # invariant: > 0
  max_factor      : real         # invariant: > 0 and >= min_factor
  base_velocity   : real         # invariant: > 0; the angular speed that means "pitch 1"
  smoothed_speed  : real         # initialized to base_velocity, not to zero
  sound           : sound handle # looping, positional
  previous_pose   : transform    # the bone's world transform last frame
  skeleton        : reference to the animated model
```

**Invariants** — the three tuning numbers come from configuration and are validated on
load; a configuration that violates them is an authoring error, reported by name so the
data can be fixed.

The smoothed speed starts at the reference speed rather than at zero. Starting at zero
would make every object spend its first second of existence ramping up through the
threshold and briefly sounding its loop.

## Constants

```text
smoothing    = 0.99   # weight kept from the previous frame
threshold    = 0.2    # smoothed angular speed at which the loop starts and stops
```

The smoothing weight is extreme — ninety-nine parts history to one part measurement — and
it is not frame-rate compensated. A rebuild running at a different frame rate will smooth
over a different wall-clock window and must convert this to a time constant to sound the
same.

## `update`

**Contract** — one frame's work. Computes the bone's current world transform, differences
it against last frame's to get linear and angular velocity, smooths the angular magnitude,
starts the loop if it is not playing and the speed is above threshold, sets the loop's
pitch and position, and requests a deferred stop when the speed falls back below threshold.
Returns nothing; touches the audio device. Must be called with a valid sound handle.

```text
FUNCTION update(time_delta, object)
  pose := object world transform composed with the bone's local-to-model transform
  (linear, angular) := differentiate(previous_pose, pose, time_delta)
  smoothed_speed := smoothed_speed*smoothing + magnitude(angular)*(1 - smoothing)

  IF the loop is not sounding
    IF smoothed_speed > threshold THEN play(object)
    ELSE RETURN                     # nothing to update on a silent player

  factor := smoothed_speed / base_velocity
  pitch := 1
  IF factor > max_factor THEN pitch := max_factor
  IF factor < min_factor THEN pitch := min_factor

  set the loop's pitch and its world position (the bone's origin)

  IF smoothed_speed < threshold THEN stop the loop after the current buffer
  previous_pose := pose
```

**Notes** — the pitch rule is not a clamp, though the source's commented-out line shows a
clamp was intended. As written, a factor *inside* the authored band yields pitch exactly 1,
and only a factor outside the band moves the pitch — to the bound it exceeded. So the
band is a dead zone, not a range: the sound plays at reference pitch across the whole
normal operating speed and jumps to a fixed higher or lower pitch outside it. Whether this
is the intended effect or an unfinished clamp is not recoverable from the source; a rebuild
reproducing the original's sound must copy it as written, and one that clamps instead will
sound smoother and different.

The stop is *deferred*: it lets the currently queued audio finish rather than cutting. A
hard stop at the threshold clicks.

Starting the loop resets the previous transform to the current one, so the first frame
after starting measures a zero difference rather than the accumulated drift of however long
the player was silent.

## `play` / `stop`

**Contract** — `play` reseeds the previous transform from the current pose and starts the
sound looping, positioned at the object. `stop` stops it immediately. Both require the
sound handle to exist.

**Notes** — `play` positions the source at the *object's* origin while `update` positions
it at the *bone's*. The discrepancy lasts one frame and is invisible.

## `is_active`

**Contract** — answers true unconditionally.

**Notes** — the real test, asking the sound source whether it is still sounding, is
commented out in the source. As it stands, a player that exists is considered active
forever. The free form `is_active(player)` adds only a null test, so the whole query
reduces to "does this object have a motion sound player at all". A rebuild should either
implement the real test or drop the query.

## `create_moving_bones_snd_player`

**Contract** — the factory. Given a game object, finds its animated model, and looks for a
configuration section named after the feature in **two** places in order: the object's own
spawn configuration first, then the model's embedded user data. Returns a new player if
either declares the section, and nothing if neither does. Never fails on an object that
simply does not use the feature.

```text
FUNCTION create(object) -> optional<MovingBonesSoundPlayer>
  skeleton := the object's animated model
  result := try_create(object's spawn configuration, skeleton, object transform)
  IF result exists THEN RETURN result
  RETURN try_create(the model's embedded user data, skeleton, object transform)

FUNCTION try_create(config, skeleton, transform) -> optional<...>
  IF config is absent OR has no section "moving_bones_snd_player" THEN RETURN none
  RETURN a new player loaded from that section
```

**Invariants** — the two-source lookup order is load-bearing: a per-object override in the
spawn record must win over the shared default carried inside the model file, so that one
model can be reused with different sounds.

## `load`

**Contract** — reads the section: the sound resource name (created as a positional effect
source), the bone name (resolved to an index against the skeleton), and the three tuning
numbers. Validates the invariants above. Seeds the smoothed speed to the reference speed
and the previous transform to the object's transform at construction.

**Notes** — the sound is created as an *effect*-class, positional source, which places it
under the effects volume and inside the 3D mixer rather than the ambient bed. Machinery
that is meant to be heard at distance depends on that classification.
