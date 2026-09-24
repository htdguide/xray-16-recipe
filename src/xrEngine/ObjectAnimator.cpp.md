# src/xrEngine/ObjectAnimator.cpp

> Plays an authored rigid-body motion — position and orientation over time — for anything that moves without a skeleton.

**Needs** — [`ObjectAnimator.h`](ObjectAnimator.h.md) · [`xrCore/Animation/Motion.hpp`](../xrCore/Animation/Motion.hpp.md) · [`xrCore/FS.h`](../xrCore/FS.h.md)
**Used by** — [`ObjectAnimator.h`](ObjectAnimator.h.md)
**Tier floor** — T2: curve evaluation and a transform build, once per animated object per frame

## Purpose

Skeletal animation drives creatures. This drives everything else that follows an authored
path: cameras in scripted scenes, doors, lifts, the camera rail of a cutscene, a level's
moving props. One of these owns a bank of named motions — each a translation curve and a
rotation curve over the same timeline — and plays exactly one, producing a single
transform the owner then applies.

It is separate from the skeletal animation system because it needs none of it: no bones,
no blending, no bank sharing. The whole animator is one motion pointer and a playback
cursor.

## State

```text
RECORD ObjectAnimator
  name     : text               # the file this bank was loaded from
  motions  : list<Motion>       # sorted by name; owned
  current  : optional<Motion>   # the one playing; none when stopped
  params   : PlaybackCursor     # frame position, playing/paused flags
  speed    : real               # playback multiplier, 1.0 = authored rate
  loop     : bool
  xform    : matrix4            # the result: last evaluated pose

# invariant: motions is sorted by name, so lookup is a binary search
# invariant: xform is identity whenever current is none
# invariant: params always describes current; changing current resets it
```

## `Load`

**Contract** — loads a bank from a named file, replacing whatever was loaded, and leaves
the animator stopped. The file is searched in the current level's directory first and the
shared animation directory second, so a level may override a shared motion by shipping its
own copy under the same name. A name found in neither is a fatal error — a missing motion
means a scripted sequence will silently not happen, which is worse than stopping.

Two container shapes are accepted, distinguished by extension: a *single* motion file and
a *bank* file whose body is a count followed by that many motions. Either way the result
is sorted by name so `Play` can binary-search.

```text
FUNCTION load(animator, name)
  path = resolve(name) IN level_directory, ELSE shared_animation_directory
  IF path NOT FOUND
    FAIL WITH "can't find motion file", name
  clear(animator)
  IF extension IS single-motion
    m = read one motion from path
    IF NOT m.ok
      FAIL WITH "incorrect file version"
    motions = [m]
  ELSE IF extension IS bank
    f = open(path)
    count = f.read int
    REPEAT count TIMES
      m = read one motion from f
      IF NOT m.ok
        FAIL WITH "incorrect file version"
      motions.append(m)
  sort motions by name
```

**Notes** — a version mismatch is fatal rather than skipped. The animation container
carries no forward compatibility: an unrecognised version means the curve data would be
misread, and a misread motion teleports whatever it drives.

## `Play`

**Contract** — starts a motion by name, or the first motion in the bank when no name is
given. An unknown name is fatal, for the same reason a missing file is. Starting a motion
resets the playback cursor to the motion's start, sets the loop flag, and identity-clears
the output transform until the first update.

```text
FUNCTION play(animator, loop, name) -> Motion
  IF name IS given
    m = binary search motions FOR name
    IF NOT FOUND
      FAIL WITH "cycle not found", name
  ELSE
    IF motions IS empty
      FAIL WITH "cycle not found"
    m = motions.first
  animator.loop = loop
  set_active(animator, m)        # resets the cursor, identity-clears xform
  animator.params.play()
  RETURN m
```

**Notes** — "cycle" is the project's word for one named motion inside a bank.

## `Update`

**Contract** — advances the cursor by the elapsed seconds scaled by `speed`, wrapping when
looping and clamping when not, and rebuilds the output transform. Does nothing when
stopped. The caller supplies the delta, so an animator on a scheduled object advances at
that object's rate, not the frame rate.

```text
FUNCTION update(animator, dt_seconds)
  IF animator.current IS none
    RETURN
  (position, rotation) = animator.current.evaluate_at(animator.params.frame)
  animator.params.advance(dt_seconds, animator.speed, animator.loop)
  animator.xform = rotation_from_euler(rotation.x, rotation.y, rotation.z)
  animator.xform.set_translation(position)
```

**Invariants** — the pose is evaluated at the cursor's *current* position and the cursor is
advanced afterwards, so the transform an owner reads this frame corresponds to the time at
the start of the frame, not the end. Every consumer is written against that one-frame
convention; changing it shifts every scripted camera by a frame.

**Notes** — rotation arrives as three Euler angles and is composed in a fixed axis order.
That order is part of the authored data's meaning: the same three numbers composed in a
different order give a different orientation, and the shipped `.anm` files were exported
against this one.

## `Stop` / `Pause` / `GetLength`

**Contract** — `Stop` drops the current motion and identity-clears the transform; a stopped
animator produces no motion at all rather than freezing on its last pose. `Pause` freezes
the cursor without dropping the motion, so resuming continues. `GetLength` reports the
current motion's duration in seconds (frames over authored rate, ignoring `speed`), and
zero when stopped.

## `Clear`

**Contract** — destroys the whole bank and stops. Called by `Load` before loading and at
destruction.

## `DrawPath`

**Contract** — editor-only. Samples the current motion's translation curve at a fixed 30 Hz
regardless of the motion's authored rate and draws it as a polyline, then marks each
authored key with a cross and its time, but only for keys within a fixed distance of the
editor camera so a long path does not fill the screen with text.

**Notes** — the 30 Hz sampling rate and the distance cutoff are display choices with no
effect on playback; a rebuild may pick its own.
