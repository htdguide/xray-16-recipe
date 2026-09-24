# src/xrCore/Animation/SkeletonMotions.cpp

> Loads an animation bank out of a model file without copying a single key, and shares what it can.

**Needs** — [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md) · [`SkeletonMotionDefs.hpp`](SkeletonMotionDefs.hpp.md) · [`Motion.hpp`](Motion.hpp.md) · [`../FMesh.hpp`](../FMesh.hpp.md) · [`../FS.h`](../FS.h.md) · [`../LocatorAPI.h`](../LocatorAPI.h.md) · [`../xr_ini.h`](../xr_ini.h.md) · [`../xrsharedmem.h`](../xrsharedmem.h.md) · [`../crc32.cpp`](../crc32.cpp.md) · [`Include/xrRender/Kinematics.h`](../../Include/xrRender/Kinematics.h.md)
**Used by** — [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md)
**Tier floor** — T1: every key array in the result is a pointer into the reader's buffer, which for an archived model is a memory-mapped region.

## Purpose

Turns two chunks of a model file into a playable animation bank. The data model is in [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md); what is here is the two-pass load, the bone remapping it depends on, and the sharing that makes it affordable.

The defining decision: **nothing is copied**. The key arrays in the loaded bank point directly into the reader's bytes. For a model served from an archive uncompressed, that is the mapped file — so loading a hundred-megabyte animation set costs page faults, not allocations. It also means the bank's lifetime is tied to the reader's, which is why banks are reference-counted rather than owned by one model.

## The two chunks

**Frozen.** Both live in a model file ([`FMesh.hpp`](../FMesh.hpp.md)).

```text
# chunk 15 — motion parameters. Read FIRST; its absence is fatal.
RECORD MotionParams
  version      : int (16-bit)          # at most 4; a higher value is refused
  part_count   : int (16-bit)
  REPEAT part_count TIMES
    part_name  : text (zero-terminated)
    bone_count : int (16-bit)
    REPEAT bone_count TIMES
      bone_name        : text (zero-terminated)
      remap_slot       : int (32-bit)   # see The remap, below
  motion_count : int (16-bit)
  REPEAT motion_count TIMES
    motion_name : text (zero-terminated)
    flags       : int (32-bit)
    ... the MotionDefinition fields (see SkeletonMotions.hpp)
    IF version >= 4 THEN
      mark_count : int (32-bit)
      REPEAT mark_count TIMES: a MotionMark
```

```text
# chunk 14 — the motions themselves. A container of nested chunks.
#   sub-chunk 0        : the motion count, as a 32-bit integer
#   sub-chunk n+1      : motion number n
RECORD MotionChunk
  name         : text (zero-terminated)
  sample_count : int (32-bit)
  REPEAT (number of bones) TIMES        # IN THE SKELETON'S ORDER, remapped
    flags      : int (8-bit)
    IF flags HAS rotation_absent THEN
      rotation : RotationKey            # exactly one, inline, NO checksum
    ELSE
      crc      : int (32-bit)           # content hash of the array that follows
      rotations: RotationKey[sample_count]
    IF flags HAS translation_present THEN
      crc      : int (32-bit)
      keys     : TranslationKey16[sample_count]  if flags HAS translation_is_16
                 TranslationKey8 [sample_count]  otherwise
      size     : real[3]
      init     : real[3]
    ELSE
      init     : real[3]                # the constant position
```

**Invariants** — the per-bone records in a motion chunk appear in the **file's** bone order, and the remap table from chunk 15 converts that to the skeleton's order. The two are not the same and cannot be assumed to be.

The checksum precedes its array and is absent for the single-rotation case, where the loader computes a hash of the one key itself so the sharing table still has a key to use.

The motion count is stored twice: once in chunk 15 as the definition count, and once in chunk 14's sub-chunk 0. They must agree, and the sub-chunk numbering — motion *n* lives in sub-chunk *n+1* — means sub-chunk 0 is free to hold the count.

## `motions_value.load`

```text
FUNCTION load_bank(name, model_reader, skeleton_bones) -> bool
  # --- pass one: parameters -------------------------------------------------
  params := open_chunk(model_reader, 15)
  IF params IS none THEN FAIL WITH "old skinned model version unsupported"
  version := read_u16(params)
  FAIL IF version > 4

  remap := array of (bone count) entries, all "no bone"
  covered := 0
  FOR EACH part IN parts
    part.name := lowercase(read_cstring(params))
    FOR EACH entry IN part.bones
      bone_name  := read_cstring(params)
      remap_slot := read_u32(params)
      entry      := index of bone_name IN skeleton_bones
      FAIL IF entry IS "no bone"              # motion names a bone the model lacks
      remap[remap_slot] := entry
    covered := covered + count(part.bones)
  FAIL IF covered != count(skeleton_bones)    # partition must cover every bone

  FOR EACH motion index i
    motion_name := lowercase(read_cstring(params))
    flags       := read_u32(params)
    definitions[i] := read_motion_definition(params, flags, version)
    IF flags HAS effect THEN effects[motion_name] := i
    ELSE                     cycles [motion_name] := i
    motion_map[motion_name] := i
  close(params)

  # --- pass two: the keys ---------------------------------------------------
  data := open_chunk(model_reader, 14)
  IF data IS none THEN RETURN false
  motion_count := read_chunk(data, 0) AS a 32-bit integer
  FAIL IF motion_count >= 16384           # 14 bits; the top 2 are the partition

  FOR EACH bone IN skeleton_bones
    motions[bone.name] := array of motion_count BoneMotions    # transposed

  FOR i FROM 0 TO motion_count - 1
    seek to sub-chunk i + 1 IN data
    motion_name  := read_cstring(data)
    sample_count := read_u32(data)
    FOR file_index FROM 0 TO count(skeleton_bones) - 1
      bone := skeleton_bones[remap[file_index]]
      m    := motions[bone.name][i]
      m.sample_count := sample_count
      m.flags        := read_u8(data)

      IF m.flags HAS rotation_absent THEN
        key  := the RotationKey at the CURRENT POSITION, not copied
        hash := checksum(those 8 bytes)
        m.rotations := share(hash, count: 1, at: current position)
        advance(data, 8)
      ELSE
        hash := read_u32(data)
        m.rotations := share(hash, count: sample_count, at: current position)
        advance(data, sample_count * 8)

      IF m.flags HAS translation_present THEN
        hash := read_u32(data)
        IF m.flags HAS translation_is_16 THEN
          m.trans16 := share(hash, sample_count, at: current position)
          advance(data, sample_count * 6)
        ELSE
          m.trans8  := share(hash, sample_count, at: current position)
          advance(data, sample_count * 3)
        m.size := read_vec3(data)
        m.init := read_vec3(data)
      ELSE
        m.init := read_vec3(data)
  close(data)
  RETURN true
```

**Invariants** — `share(hash, count, at)` consults a process-wide table keyed by the hash: an array already registered under that hash is reused and the reader's bytes are ignored; otherwise the pointer is registered. That is the fine-grained sharing, and it is why identical bone animations across motions and across models cost nothing after the first.

Advancing by `sample_count * width` immediately after taking the pointer is what keeps the cursor and the pointer consistent. The width per key form — 8, 6 and 3 bytes — is frozen and must match the records in [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md).

**Notes** — the tools' build tolerates a missing bone and reports it; the game's build treats it as fatal. That split exists so an author can open a broken rig and see what is wrong, while the game refuses to run with animation it cannot map.

The remap table is indexed by the *file's* slot, which is read from the file, not derived. So a model may list its bones in any order in chunk 14 as long as chunk 15 says what that order is.

A stale check in the parameters pass verifies, in a diagnostic build only, that the motion at sub-chunk *n+1* has the name the parameter table gave index *n*. That check is the only thing tying the two chunks' orderings together, and a rebuild should make it unconditional.

## `CMotionDef.Load`

**Contract** — read the seven playback fields, quantizing the four rate-like reals on the way in, then apply the cycle correction. From version 4 onward, read the motion's marks.

```text
FUNCTION read_motion_definition(r, flags, version) -> MotionDefinition
  d.bone_or_part := read_u16(r)
  d.motion       := read_u16(r)
  d.speed   := quantize(read_real(r))
  d.power   := quantize(read_real(r))
  d.accrue  := quantize(read_real(r))
  d.falloff := quantize(read_real(r))
  d.flags   := low 16 bits of flags
  IF d IS NOT an effect AND d.falloff >= d.accrue THEN
    d.falloff := d.accrue - 1                 # see SkeletonMotions.hpp
  IF version >= 4 THEN read the marks
```

**Notes** — the correction subtracts one *quantized unit*, which is about 0.0015 in the decoded range. It is the smallest possible fix, chosen to disturb authored tuning as little as possible while restoring the invariant.

## `CPartition.load` — the configuration override

Contracted in [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md). The path is the model's own path with its extension replaced, resolved under the meshes root.

## `motion_marks.Load` / `Save`

**Contract** — a name as a *line* (terminated by carriage return and line feed, not by a zero byte — note the asymmetry with everything else in this format), then a 32-bit interval count, then that many pairs of reals.

**Notes** — the line-terminated string is the one place in the runtime animation format that uses that convention. It is almost certainly an accident of the mark support being added later by different hands, and it is frozen.

## `motions_container`

Contracted in [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md). Three operations: ask whether a bank is loaded; dock a bank by name, loading it if absent and discarding a failed load; and sweep, either destroying everything or only the unreferenced.

**Notes** — a bank that fails to load is destroyed and *not* cached as a failure, so every model that wants it retries. For a genuinely broken bank that is a repeated cost, but it also means a bank that failed because its skeleton did not match one model can still succeed for another — which is the case that matters, since a bank is keyed by name and asked for with a particular skeleton.

## Could not recover

- How the build tool decides between the 8-bit and 16-bit translation encodings for a given bone. Only the flag recording the decision survives.
- Whether the remap slots in chunk 15 are ever non-contiguous. The loader would tolerate gaps (unfilled entries stay at the absent marker) but would then index with the absent marker and fault.
