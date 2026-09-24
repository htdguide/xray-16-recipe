# src/xrCore/Animation/Motion.cpp

> Serializes authored motions across five format versions, and reconciles a motion's bone list with a skeleton's.

**Needs** — [`Motion.hpp`](Motion.hpp.md) · [`Envelope.hpp`](Envelope.hpp.md) · [`Bone.hpp`](Bone.hpp.md) · [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md) · [`../FS.h`](../FS.h.md) · [`../LocatorAPI.h`](../LocatorAPI.h.md)
**Used by** — [`Motion.hpp`](Motion.hpp.md)
**Tier floor** — T1: several versions read field widths that differ from the current ones, so the reader must describe each layout explicitly.

## Purpose

The authored motion's persistence. The data model is in [`Motion.hpp`](Motion.hpp.md); this file is the versioned reader and writer, and the two operations — reordering and evaluation — that turn an authored motion into something a skeleton can use.

The version handling is the substance. Five versions of the skeleton motion and three of the object motion shipped, and they differ in field widths, in whether a bone is named or numbered, and in which envelope encoding is used.

## State — the container

Both kinds are stored inside a chunk whose identifier names the kind, in the container format of [`FS.cpp`](../FS.cpp.md):

| Chunk | Meaning | Current version |
|---|---|---|
| 0x1100 | an object motion | 5 |
| 0x1200 | a skeleton motion | 7 |
| 0x9000 / 0x9001 | a clip's version / data | 2 |

Every motion begins with the common preamble:

```text
RECORD MotionPreamble
  name        : text (zero-terminated)
  frame_start : int (32-bit)
  frame_end   : int (32-bit)
  fps         : real
  version     : int (16-bit)       # follows the preamble; selects what comes next
```

## `COMotion.Load` — the object motion's three versions

```text
FUNCTION load_object_motion(r)
  read the preamble; version := read_u16(r)
  SELECT version
    CASE 3:                        # six curves, unquantized key encoding
      FOR ch IN 0..5 DO load_envelope_v1(r, channel[ch])
    CASE 4:                        # quantized keys, but a DIFFERENT CHANNEL ORDER
      load_envelope_v2 INTO position_x
      load_envelope_v2 INTO position_y
      load_envelope_v2 INTO position_z
      load_envelope_v2 INTO rotation_PITCH      # <-- pitch before heading
      load_envelope_v2 INTO rotation_HEADING
      load_envelope_v2 INTO rotation_bank
    CASE 5:                        # quantized keys, canonical channel order
      FOR ch IN 0..5 DO load_envelope_v2(r, channel[ch])
    OTHERWISE: RETURN failure
```

**Invariants** — **version 4 stores pitch and heading in the opposite order to version 5.** This is the only difference between them and it is invisible unless a model rotates about both axes. A rebuilder who treats 4 and 5 as the same layout will produce animation that is correct for yaw-only and wrong for anything else.

## `CSMotion.Load` — the skeleton motion's four versions

```text
FUNCTION load_skeleton_motion(r)
  read the preamble; version := read_u16(r)

  CASE version = 4:
    bone_or_part := low 16 bits of read_u32(r)
    flags.fx          := read_u8(r) IS NON-ZERO      # two separate bytes,
    flags.stop_at_end := read_u8(r) IS NON-ZERO      # not a flag word
    speed, accrue, falloff, power := four reals
    bone_count := read_u32(r)
    FOR EACH bone
      name  := THE BONE'S INDEX RENDERED AS DECIMAL TEXT   # no name is stored
      flags := low 8 bits of read_u32(r)
      FOR ch IN 0..5 DO load_envelope_v1(r, channel[ch])

  CASE version = 5:
    flags        := low 8 bits of read_u32(r)        # now a flag word
    bone_or_part := low 16 bits of read_u32(r)       # order SWAPPED vs v4
    speed, accrue, falloff, power := four reals
    bone_count := read_u32(r)
    FOR EACH bone
      name  := read_cstring(r)                       # names appear
      flags := low 8 bits of read_u32(r)
      FOR ch IN 0..5 DO load_envelope_v1(r, channel[ch])

  CASE version >= 6:
    flags        := read_u8(r)                       # widths shrink
    bone_or_part := read_u16(r)
    speed, accrue, falloff, power := four reals
    bone_count := read_u16(r)
    FOR EACH bone
      name  := read_cstring(r)
      flags := read_u8(r)
      FOR ch IN 0..5 DO load_envelope_v2(r, channel[ch])   # quantized keys

  IF version >= 7 THEN
    mark_count := read_u32(r)
    FOR EACH mark DO load_mark(r)                    # see SkeletonMotions.cpp

  lowercase every bone name
```

**Invariants** — version 4 has *no bone names at all*: a bone is identified by its position in the list, and the loader fabricates a name from the index so the rest of the code has something to match on. That only works because such a motion is always paired with the skeleton it was authored against. Reordering it against a different skeleton is not possible.

Versions 4 and 5 swap the order of the flag word and the bone-or-partition field. Both are read as 32 bits and truncated.

A version the loader does not recognize — anything below 4 — falls through **leaving the motion at its constructed defaults and reporting success**. That is a defect: a caller cannot tell an empty motion from an unsupported one.

**Notes** — the marks chunk was added in version 7 and is appended after everything else, which is what makes the addition backward-compatible: an older reader stops at the end of the bone list and never sees it.

## `CSMotion.Save`

**Contract** — always writes the current version (7). Lowercases each bone name on the way out. Writes the flag word as a *signed* byte, which is a transcription slip with no effect since only eight flags exist.

## `SortBonesBySkeleton`

**Contract** — reorder a motion's bone list so its *n*-th entry is the *n*-th bone of a given skeleton, inventing an entry for any skeleton bone the motion does not mention. This is what makes a motion authored against one rig playable on another with the same bone names.

```text
FUNCTION reorder(motion, skeleton_bones)
  result := []
  FOR EACH bone IN skeleton_bones                    # skeleton order wins
    bm := motion.bone_motions WHERE name = bone.name
    IF bm IS none THEN
      bm := new BoneMotion(name: bone.name)
      bm.flags := motion.bone_motions[0].flags       # copy the first entry's
      FOR ch IN 0..5 DO bm.channels[ch] := empty envelope
      # seed each channel with a single key holding the bone's REST pose
      insert_key(bm.position_x, at 0, bone.rest_offset.x)
      insert_key(bm.position_y, at 0, bone.rest_offset.y)
      insert_key(bm.position_z, at 0, bone.rest_offset.z)
      insert_key(bm.rotation_heading, at 0, bone.rest_rotation.x)   # SEE NOTE
      insert_key(bm.rotation_pitch,   at 0, bone.rest_rotation.y)
      insert_key(bm.rotation_bank,    at 0, bone.rest_rotation.z)
    result := result + bm
  motion.bone_motions := result
```

**Invariants** — a missing bone is filled with its **rest pose**, not with zero. A zero-filled bone would snap to the model origin.

**Notes** — the seeding writes the rest rotation's components into the heading, pitch and bank slots *in order* — x into heading, y into pitch, z into bank — which is **not** the mapping the evaluator uses (which reads heading into y and pitch into x). The two disagree, so a synthesized bone's rest rotation is permuted. This is a bug in the original and is only invisible because the synthesized case is rare and usually involves a bone whose rest rotation is near identity. A rebuild should use the evaluator's mapping.

An earlier version of this function refused outright when a bone was missing. The synthesis was added so third-party rigs could be animated with shipped motions.

## `_Evaluate`

**Contract** — evaluate all six channels at a time and produce a translation and a rotation. The object form evaluates the motion's own channels; the skeleton form evaluates one bone's. Both use the channel-to-component mapping contracted in [`Motion.hpp`](Motion.hpp.md).

## `NormalizeKeys`

**Contract** — retime a range of an object motion so the object moves at a constant *spatial* speed rather than a constant temporal one. For each key inside the range, the arc length travelled since the previous key is measured by sampling the position curves at a fine step and summing the distances; the key's new time is that length divided by the requested speed. Keys after the range keep their spacing and slide.

**Invariants** — every channel must end up with the same number of keys as the position channel, because the new times are applied to all six by index. That holds only if the motion was authored with synchronized keys, which the tools guarantee.

**Notes** — the arc length is sampled at a very small fixed step rather than at the frame rate, which makes this quadratic in the motion's length and is why it is an editor operation and not a runtime one.

## `Optimize`

**Contract** — run the constant-channel simplification from [`Envelope.cpp`](Envelope.cpp.md) on every channel of every bone. This is what shrinks a shipped animation bank.

## `CClip.Save` / `Load`

**Contract** — a two-chunk container: a version chunk and a data chunk holding the clip's name, its four animation slots as (name, index) pairs, the effect slot, the effect strength and the length. A version mismatch is a refusal, not an adaptation — there is only one version and the clip format is editor-internal.
