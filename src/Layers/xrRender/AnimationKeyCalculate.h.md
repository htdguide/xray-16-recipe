# src/Layers/xrRender/AnimationKeyCalculate.h

> The whole of the per-bone animation mathematics: how a quantized key becomes a pose, how several poses become one, and how the four channels fold together.

**Needs** — [`Animation.h`](Animation.h.md) · [`xrCore/Animation/SkeletonMotions.hpp`](../../xrCore/Animation/SkeletonMotions.hpp.md) · [`Include/xrRender/animation_blend.h`](../../Include/xrRender/animation_blend.h.md)
**Used by** — [`SkeletonAnimated.cpp`](SkeletonAnimated.cpp.md)
**Tier floor** — T1: this is the hottest loop in the animation system — it runs once per bone per running animation per frame, for every animated model in the scene — and it reads the shipped quantized key arrays as memory images with a modulo index. A rebuild may parse the keys into a decoded form at load time and rise to T2, at the cost of several times the memory for the animation bank.

## Purpose

A skinned model's pose for this frame is built bone by bone. For each bone the system must: pull a rotation and a translation out of each running animation's compressed key stream at this animation's current time; combine the several animations running on one channel into one pose; and combine the four channels into the bone's final pose. This file is all of that, and nothing else. It is a header with no implementation file because every function in it is called from the skeleton's inner loop and must inline.

## The compressed key format, and why it decodes this way

Animation data ships quantized. Reproducing the dequantization exactly is a conformance requirement, not a quality choice: the same bytes must produce the same pose.

```text
RECORD KeyRotation            # 8 bytes, four signed 16-bit components
  x, y, z, w : int (16-bit, signed)
  # a quaternion, each component scaled by a fixed reciprocal constant.
  # NOT normalized on read: the quantization error is small enough that the
  # subsequent slerp absorbs it, and normalizing every key costs a square root
  # per bone per blend per frame.

RECORD KeyTranslation8        # 3 bytes
  x, y, z : int (8-bit)
RECORD KeyTranslation16       # 6 bytes
  x, y, z : int (16-bit)
  # both decode as  component * motion.scale + motion.origin,
  # where scale and origin are per-motion, per-axis, stored with the motion.
```

A motion carries three flags that select among four shapes:

- **rotation absent** — the bone does not rotate in this motion; key 0 is the constant rotation, and no interpolation happens at all.
- **translation present** — when clear, the bone does not move and the motion's origin *is* the translation. Most bones in most motions are in this state, which is why the flag exists.
- **translation is 16-bit** — chooses which of the two translation records the stream holds. Authored per motion by the exporter based on the motion's spatial extent.

**Invariants**

- Sampling rate is fixed at 30 frames per second for every motion in the game. The current time is multiplied by it to get a fractional frame index; the integer part indexes the key and the fractional part is the interpolation weight.
- Key indices wrap modulo the motion's key count. This is what makes a looping animation loop without a special case, and it means a motion's last key interpolates back toward its first — so an authored non-looping motion must repeat its first pose at the end, or it will visibly snap. The shipped data honours this.
- Rotation interpolates by shortest-arc spherical interpolation, translation by straight linear interpolation. The weight is clamped into the unit range even though it is derived from a fractional part that cannot leave it — floating-point accumulation of the clock can push it a hair outside and the clamp is load-bearing.

## `dequantize(key_out, blend, motion)`

**Contract** — reads one bone's pose out of one motion at the blend's current time. No allocation, no locking; called from several worker threads at once against immutable motion data, and *that immutability is the only thing making it thread-safe*.

```text
FUNCTION dequantize(out, blend, motion) 
  time  = blend.time_current * 30              # samples per second
  frame = floor(time)
  delta = time - frame
  count = motion.key_count

  IF motion.rotation_absent
    out.rotation = decode_rotation(motion.rotations[0])
  ELSE
    a = decode_rotation(motion.rotations[frame       MOD count])
    b = decode_rotation(motion.rotations[(frame + 1) MOD count])
    out.rotation = slerp(a, b, clamp(delta, 0, 1))

  IF motion.translation_present
    a = decode_translation(motion, frame       MOD count)
    b = decode_translation(motion, (frame + 1) MOD count)
    out.translation = lerp(a, b, delta)
  ELSE
    out.translation = motion.origin
```

## `mix_interpolate(result, poses, blends, count)`

**Contract** — folds several poses running on *one* channel into one, weighted by each running animation's current blend amount. Never allocates; the working array is stack-sized by the blend ceiling.

```text
FUNCTION mix_interpolate(result, poses, blends, count)
  IF count = 0    -> result is the ZERO pose, not identity     # see note
  IF count = 1    -> result = poses[0]
  IF count = 2    -> result = key_interp(poses[0], poses[1], w1 / (w0 + w1))
  OTHERWISE
    sort (pose, weight) pairs by DECREASING weight
    running = weight[0] ; result = pose[0]
    FOR EACH remaining pair
      running = running + pair.weight
      result  = key_interp(result, pair.pose, pair.weight / running)
```

**Invariants**

- The zero-count result is an all-zero quaternion and a zero translation, which is *not* the identity pose. It is a deliberate sentinel: a bone with no animation touching it must be recognizable downstream as "nothing wrote me" so the caller can substitute the bind pose. A rebuild that helpfully returns identity here will silently plant every unanimated bone at the model origin.
- The running-weight fold is an incremental weighted average and is correct for any count, but it is *order dependent* in floating point. Sorting by decreasing weight is what makes it stable: the dominant pose is established first and the small contributions perturb it, rather than the reverse. The sort comparator is deliberately inverted for this reason and is the only reason the sort exists.
- Every divisor is guarded against a zero weight sum and yields a weight of zero — meaning "keep what you have" — rather than a non-finite value. A blend whose amount has decayed to exactly zero is a normal state, reached every time an animation finishes falling off.

## `mix_channels(result, poses, channel_defs, count)`

**Contract** — folds the four channels' finished poses into the bone's final pose, using each channel's extern rule.

```text
FUNCTION mix_channels(result, poses, defs, count)
  result = poses[0]                  # channel 0 is the base; its rule is not consulted
  lerp_weight_total = 0
  FOR i FROM 1 TO count - 1
    IF defs[i].rule.extern = add
      result = key_mad(result, poses[i], defs[i].factor)     # layered delta
    ELSE   # lerp
      lerp_weight_total = lerp_weight_total + defs[i].factor
      result = key_interp(result, poses[i], defs[i].factor / lerp_weight_total)
```

**Invariants** — The same incremental-average trick as above, but *without* a sort, because the channels have a fixed meaning and their order is the order they must be applied in. Channel 0 is always the base and is copied in wholesale.

## The pose algebra

**Contract** — the small closed set of operations poses are combined with. All are pure, allocate nothing, and operate on a (rotation, translation) pair treated as a rigid transform.

```text
key_identity(k)                 -> identity rotation, zero translation
key_add(res, a, b)              -> rotations composed, translations summed
key_sub(res, a, b)              -> a composed with b inverted; translations subtracted
key_scale(res, k, v)            -> rotation angle scaled by v about its own axis;
                                   translation scaled by v
key_mad(res, a, b, v)           -> key_add(res, a, key_scale(b, v))
key_interp(res, a, b, t)        -> slerp the rotations, lerp the translations
keys_subtract(poses, base, n)   -> make a run of poses relative to a base pose
```

**Invariants**

- Scaling a rotation means decomposing it to axis and angle and scaling the *angle*. That is the only definition under which "half of this rotation" means what an animator expects, and it is why additive channels can be weighted at all. It costs a trigonometric decomposition per scaled bone, which is the price of the feature.
- Composition is *right*-handed throughout: "add right" and "sub right" in the original's own words — the second operand is applied in the first's frame. Reversing it mirrors every additive animation in the game.
- Quaternions are deliberately **not** normalized after composition. The commented-out normalizations in the original are not oversights; each was removed because the error stays bounded across the few compositions one bone sees, and the square root is measurable at this call frequency. A rebuild that composes more deeply than four levels should re-add it.

**Notes** — `keys_subtract` exists for one purpose: turning an absolutely-authored motion into a delta relative to a reference pose, so it can be played on an additive channel. The game data ships some motions that were authored absolutely and are used additively.

## `check_scale(transform)`

**Contract** — a guard, not a computation: reports whether a transform's determinant lies in a narrow band around one.

**Notes** — The band is roughly 0.8 to 1.3, which is far wider than floating-point drift and far narrower than a real scale. It is catching one specific failure: a quaternion that has been composed into non-unit length by a chain of unnormalized multiplications, or a bone matrix that an external system (physics, inverse kinematics) has written garbage into. The asymmetry of the band — more room above one than below — has no recoverable justification; it looks like it was widened once to stop a false positive and never re-derived.

## `mix_factors(weights, count)`

**Contract** — normalizes a run of weights to sum to one, in place. Unguarded against a zero sum, unlike everything else in this file — its callers have already established at least one non-zero weight.
