# src/xrCore/Animation/SkeletonMotions.hpp

> The runtime animation format: quantized keyframe arrays, shared between every object that uses the same bank, plus the partition and motion-definition tables that drive blending.

**Needs** — [`SkeletonMotions.cpp`](SkeletonMotions.cpp.md) · [`SkeletonMotionDefs.hpp`](SkeletonMotionDefs.hpp.md) · [`Bone.hpp`](Bone.hpp.md) · [`../_quaternion.h`](../_quaternion.h.md) · [`../xrsharedmem.h`](../xrsharedmem.h.md) · [`../xrstring.h`](../xrstring.h.md)
**Used by** — [`KinematicsAnimated.h`](../../Include/xrRender/KinematicsAnimated.h.md) · [`animation_blend.h`](../../Include/xrRender/animation_blend.h.md) · [`AnimationKeyCalculate.h`](../../Layers/xrRender/AnimationKeyCalculate.h.md) · [`SkeletonAnimated.cpp`](../../Layers/xrRender/SkeletonAnimated.cpp.md) · [`SkeletonAnimated.h`](../../Layers/xrRender/SkeletonAnimated.h.md) · [`xrRender_console.cpp`](../../Layers/xrRender/xrRender_console.cpp.md) · [`Motion.cpp`](Motion.cpp.md) · [`Motion.hpp`](Motion.hpp.md) · [`SkeletonMotionDefs.hpp`](SkeletonMotionDefs.hpp.md) · [`SkeletonMotions.cpp`](SkeletonMotions.cpp.md) · [`HudItem.cpp`](../../xrGame/HudItem.cpp.md) · [`WeaponKnife.cpp`](../../xrGame/WeaponKnife.cpp.md)
**Tier floor** — T1: the key arrays are pointed at mapped archive bytes and never copied, and their records are frozen packed layouts.

## Purpose

What the engine actually plays. An authored motion ([`Motion.hpp`](Motion.hpp.md)) is six spline curves per bone; a *runtime* motion is six quantized numbers per bone per sample, resampled to a fixed rate. The conversion happens when the model is built and is one-way.

Two problems shape everything here. The first is **size**: a creature has a hundred bones and hundreds of motions, so the per-sample record has to be tiny — hence the quantization and the two translation encodings. The second is **sharing**: dozens of creature models use the same animation bank, and many bones in many motions are *identical* (a bone that never moves during a motion), so the key arrays are reference-counted by content hash and the whole bank is reference-counted by name.

This header carries the data model; [`SkeletonMotions.cpp`](SkeletonMotions.cpp.md) carries the loading.

## State — the key records

**Frozen.** Two-byte packing; these records are pointed at directly inside a mapped archive.

```text
RECORD RotationKey               # 8 bytes
  x, y, z, w : int (16-bit, signed)
# A quaternion, each component scaled by the quantizer (32767) — divide by it
# to recover a value in [-1, +1].

RECORD TranslationKey8           # 3 bytes
  x, y, z : int (8-bit, signed)

RECORD TranslationKey16          # 6 bytes
  x, y, z : int (16-bit, signed)

# Both translation forms are decoded the same way: the stored component is
# scaled by the motion's own per-axis SIZE and offset by its per-axis INIT:
#     component = stored * size[axis] + init[axis]
# `size` and `init` are stored once per bone per motion, so each bone gets its
# own bounding box and the 8-bit form still has useful resolution.
```

```text
RECORD BoneMotion                # one bone's animation within one motion
  flags       : int (8-bit)
  sample_count: int (24-bit)     # packed with flags into ONE 32-bit word
  rotations   : shared array<RotationKey>
  trans8      : shared array<TranslationKey8>     # at most one of these
  trans16     : shared array<TranslationKey16>    # is populated
  init        : real[3]          # translation offset
  size        : real[3]          # translation scale; absent when the
                                 # translation is constant

# length in seconds = sample_count / 30

ENUM BoneMotionFlag
  translation_present = bit 0    # otherwise the bone holds `init` forever
  rotation_absent     = bit 1    # the bone has ONE rotation key, not an array
  translation_is_16   = bit 2    # otherwise the translation keys are 8-bit
```

**Invariants** — the flag byte and the sample count share one 32-bit word, count in the low 24 bits. That caps a motion at 16.7 million samples, which at the fixed rate is about six days. The packing is an in-memory layout choice, not a file one; the file stores them separately.

`rotation_absent` is a misnomer worth naming: it does not mean the bone has no rotation, it means the bone has exactly **one** rotation key which applies for the whole motion. The array is still allocated, with one element.

When translation is absent, `init` holds the bone's constant position and `size` is not read.

**Notes** — the choice between 8- and 16-bit translation is made per bone per motion at build time, presumably by measuring whether 8 bits of the bone's own range is enough. That decision is not recoverable from this code; only the flag that records it is.

Making the *rotation* the thing that is always present, and the translation optional, is the right way round: most bones in most motions rotate and do not translate, because translation is the parent's job in a hierarchy.

## `motion_marks` — named time intervals

```text
RECORD MotionMark
  name      : text
  intervals : list<(start, end)>     # seconds, sorted by start
```

**Contract** — a named set of time ranges within a motion, used to fire events at the right moment: footfalls, weapon-fire windows, the frames during which a melee attack can connect. Three queries: which interval (if any) contains a time; whether any interval overlaps a span, which is how a caller that advances by a frame asks "did I cross a mark"; and how far forward to the next interval's start.

**Invariants** — intervals are stored sorted by start and each interval's start is no later than its end. The containment query relies on the sorting to stop early.

**Notes** — the overlap query's cases are written out exhaustively rather than as one interval-overlap test, and the exhaustive form has an inclusive boundary at both ends, so a mark that starts exactly at the query's end counts as crossed. That inclusivity is what makes footsteps fire exactly once per step rather than zero or twice.

## `CMotionDef` — the playback parameters

```text
RECORD MotionDefinition
  bone_or_part : int (16-bit)   # a partition index for a cycle, a bone index
                                # for an effect
  motion       : int (16-bit)   # which motion in the bank
  speed        : int (16-bit)   # quantized
  power        : int (16-bit)   # quantized
  accrue       : int (16-bit)   # quantized — blend-in rate
  falloff      : int (16-bit)   # quantized — blend-out rate
  flags        : int (16-bit)   # the motion flags from Motion.hpp
  marks        : list<MotionMark>

# quantization:  stored = clamp(floor(value * 655.35), 0, 65535)
#                value  = stored / 655.35
# so the representable range is [0, 100] with about three decimal digits.
```

**Invariants** — accrue and falloff are additionally multiplied by **1.5** on the way out. That extension factor exists because the authored range tops out at 100 and some motions needed a faster blend than the quantizer could express; rather than rescale the format, the decode was scaled. It applies to accrue and falloff only, not to speed or power.

For a cycle (not an effect), **falloff is forced strictly below accrue** at load time: if the authored falloff is greater than or equal to the accrue, it is reduced to one less. A cycle that blends out at least as fast as it blends in can be scheduled out before it is fully in, which produces a visible pop. This is a data-correction rule and a rebuild must apply it, because shipped data violates it.

**Notes** — the quantizer's denominator, 655.35, is 65535 divided by 100. So this is "a percentage stored in 16 bits", and the odd-looking constant is just that.

## `CPartition` / `CPartDef` — which bones belong to which part

```text
RECORD PartitionDefinition
  name  : text
  bones : list<int>          # bone indices

RECORD Partition
  parts : PartitionDefinition[4]      # fixed; see SkeletonMotionDefs
```

**Contract** — a skeleton is split into at most four parts, each animated by its own motion and blended independently. Looking up a part by name is a linear scan over four entries; a name with no part logs a complaint and returns the absent marker rather than failing. The populated count is how many parts have a non-empty name.

**Invariants** — the partition in the animation bank must cover **every** bone of the skeleton exactly once; the loader checks the total count and refuses a mismatch. A bone in no partition would never be animated; a bone in two would be animated twice.

### Partition override from configuration

**Contract** — a model may override its partition from a text configuration file sitting beside it: same path, extension replaced with the configuration extension, resolved under the meshes root. Sections named `part_0` through `part_3` each hold a key naming the part and one key per bone, the *key* being the bone's name. A section with any entries replaces that part's bone list entirely; an empty or absent section leaves the built-in partition alone. A configuration file with no sections at all is ignored.

**Notes** — using the configuration key as the bone name and ignoring the value is an abuse of the format, but it gives an order-preserving set of names with no duplication, which is exactly what is wanted. See [`xr_ini.cpp`](../xr_ini.cpp.md) for the format.

This override is how a modder re-partitions a shipped creature without rebuilding its model.

## `motions_value` — one loaded animation bank

```text
RECORD AnimationBank
  id            : text                          # the bank's name, the share key
  motion_map    : map<text, int>                # every motion by name
  cycles        : map<text, int>                # the looping ones
  effects       : map<text, int>                # the one-shot ones
  partition     : Partition
  definitions   : list<MotionDefinition>        # indexed by motion index
  motions       : map<text, list<BoneMotion>>   # bone name -> per-motion array
  reference_count : int

# invariant: `motions[bone]` has one entry per motion, so indexing is
#   [bone name][motion index] — bones outer, motions inner. That is the
#   opposite of how the FILE stores it (motions outer, bones inner), and the
#   transposition happens at load.
# invariant: a motion is in exactly one of `cycles` or `effects`, decided by
#   its effect flag, and in `motion_map` regardless.
# invariant: motion indices fit in 14 bits — the upper two bits of a motion
#   identifier are the partition slot — so a bank holds at most 16383 motions.
```

**Notes** — the transposition is deliberate. The *player* iterates bones for one motion, but the *blender* holds several motions at once and walks bones outermost, touching one bone's several motions together. Storing bone-major makes that the cache-friendly direction.

## `motions_container` / `shared_motions` — the sharing

**Contract** — a process-wide table maps a bank name to its loaded bank. Asking for a bank by name returns the existing one if present and loads it otherwise; each holder increments the reference count and decrements on release. A cleanup pass either destroys every bank unconditionally or destroys only those with no holders.

**Invariants** — the table must be empty when the container is destroyed; a bank outliving it is a leak and is checked.

**Notes** — the release path decrements the count and drops the local reference when it reaches zero **but does not destroy the bank**. Destruction happens only in the sweep. That is deliberate: a level transition releases every model and then immediately loads the next level, which usually wants many of the same banks, so keeping them until an explicit sweep avoids reloading.

Within a bank, the key arrays are shared again at a finer grain: each array is registered under a content hash, so two motions whose bone is identical share one array. See [`xrsharedmem.h`](../xrsharedmem.h.md).

## Exported units

- **`CKey` / `CKeyQR` / `CKeyQT8` / `CKeyQT16`** — the key records above.
- **`CMotion`** — one bone's animation in one motion, with the packed flag-and-count word.
- **`motion_marks`** — named time intervals and their three queries.
- **`CMotionDef`** — the quantized playback parameters.
- **`CPartDef` / `CPartition`** — the four-way skeleton split and its configuration override.
- **`motions_value`** — a loaded bank.
- **`motions_container`** — the process-wide bank table.
- **`shared_motions`** — a counted reference to a bank, with accessors for its maps, its partition and its definitions.
