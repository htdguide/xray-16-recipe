# src/xrCore/Animation/Bone.cpp

> Serializes a bone: which chunks carry what, which are optional, and the sign flip between authored and simulated joint limits.

**Needs** — [`Bone.hpp`](Bone.hpp.md) · [`../FS.h`](../FS.h.md) · [`../_obb.h`](../_obb.h.md)
**Used by** — [`Bone.hpp`](Bone.hpp.md)
**Tier floor** — T1: two records are written as raw memory blocks, so their in-memory layout *is* the file layout.

## Purpose

The skeleton's serialization. Two things live here that a rebuilder must get exactly right: the chunk numbering with its optional members, and the sign convention flip on joint limits that is applied on the way out and *not* on the way in.

## State — the bone chunks

**Frozen.** A bone is a chunked container in the format of [`FS.cpp`](../FS.cpp.md).

| Id | Chunk | Optional | Payload |
|---|---|---|---|
| 1 | version | no | 16-bit version tag; current value 2 |
| 2 | definition | no | three zero-terminated strings: bone name, parent name, weight-map name |
| 3 | bind pose | no | rest offset (3 reals), rest rotation as Euler triple (3 reals), rest length (1 real) |
| 4 | material | no | one zero-terminated string: the surface-material name |
| 5 | shape | no | the collision-shape record as **one raw block** |
| 6 | joint | no | joint kind (32 bits), then the three limit records as **one raw block**, then whole-joint spring and damping |
| 7 | mass | yes | mass (real), centre of mass (3 reals) |
| 8 | flags | yes | the joint flag word (32 bits) |
| 9 | breakability | yes | break force, break torque |
| 16 | friction | yes | joint friction |

**Invariants** — the chunk numbers are written in decimal in the source but the last one is 0x0010, which is **16**, not 10. Anyone re-deriving the list by counting will place it wrong.

The shape record and the limits array are written with a whole-struct write, which means **the in-memory layout is the file layout**: field order, field widths and the two-byte packing are all frozen by that decision. A rebuild must serialize them field by field to the same bytes rather than reproducing a struct layout.

The optional chunks are what version drift looks like in this format: a bone written by an older tool simply lacks chunks 7, 8, 9 and 16, and the loader leaves the constructed defaults in place. That is why the defaults matter as much as the file.

**Defaults** (what an absent chunk means): joint kind *rigid*; every limit range zero with spring and damping one; whole-joint spring and damping one; no flags; break force and torque zero; friction zero; mass ten; centre of mass at the origin; material named `default_object`; shape kind *none* with an invalidated box, a zero sphere and an invalidated cylinder.

## `SJointIKData.Export` / `Import`

**Contract** — write or read the joint description as a flat record, not as chunks. This is the form embedded in the model file's inverse-kinematics chunk, as opposed to the per-bone chunked form above.

```text
FUNCTION export_joint(w, j)
  write_u32(w, j.kind)
  FOR EACH limit IN j.limits                   # exactly three
    write_real(w, -limit.range.high)           # min, SIGN FLIPPED AND SWAPPED
    write_real(w, -limit.range.low)            # max
    write_real(w, limit.spring_factor)
    write_real(w, limit.damping_factor)
  write_real(w, j.spring_factor)
  write_real(w, j.damping_factor)
  write_u32 (w, j.flags)
  write_real(w, j.break_force)
  write_real(w, j.break_torque)
  write_real(w, j.friction)                    # ONLY when version > 0

FUNCTION import_joint(r, version) -> JointData
  j.kind := read_u32(r)
  read the three limit records AS ONE RAW BLOCK          # NOT sign-flipped
  j.spring_factor := read_real(r); j.damping_factor := read_real(r)
  j.flags := read_u32(r)
  j.break_force := read_real(r); j.break_torque := read_real(r)
  IF version > 0 THEN j.friction := read_real(r)
```

**Invariants** — **the export flips the sign and swaps low with high; the import does not.** This is not a bug: the two halves are not inverses of each other. The export writes the *physics library's* convention, where the rotation direction is opposite to the authoring tool's, and the import reads a record that is already in the in-memory convention. A rebuilder implementing "read is the inverse of write" here will invert every joint stop in the game.

The three limits are read as one raw block, so the limit record's layout — two reals of range, then spring, then damping, with two-byte packing — is frozen.

## `CBone.Save` / `Load_1` / `SaveData` / `LoadData`

**Contract** — a bone is saved in two layers. `Save` writes the version, the definition and the bind pose, then delegates to `SaveData` for everything the tools can edit. `LoadData` is the matching half and is also used on its own, to re-apply edited properties to a bone whose geometry did not change.

**Invariants** — `Load_1` refuses a version it does not recognize by returning without touching the bone, leaving it at its constructed defaults. It accepts exactly two versions, 1 and 2. **Version 1 stores the bind rotation with its first two Euler components swapped**, and the loader swaps them back. Version 2 does not. That is the only difference between the two versions, and getting it wrong rotates every bone in a first-generation model ninety degrees about the wrong axis.

Bone and parent names are lowercased immediately after reading, because skeletons are matched to motion banks by name.

**Notes** — there is a third, chunkless loader (`Load_0`) that reads the definition and bind pose as a flat record with the same version-1 Euler swap. It predates the chunked form and exists for the oldest authoring files.

## `SBoneShape.Valid`

**Contract** — report whether the shape's dimensions are non-degenerate for its kind: all three half-extents for a box, the radius for a sphere, height and radius and a non-zero axis for a cylinder. Any other kind is reported valid.

**Notes** — the axis test uses the squared length, which is both cheaper and the right test: a direction vector that rounds to zero in every component is unusable regardless of how the zero arose.

## `CBone.CopyData`

**Contract** — copy the *editable* half of one bone onto another — material, shape, joint data, mass, centre of mass — leaving name, hierarchy and bind pose alone. This is how the tools apply one bone's physical properties to its mirror image.

## `CBoneData.CalculateM2B` / `CBoneData.DebugQuery`

`CalculateM2B` is contracted in [`Bone.hpp`](Bone.hpp.md), including why the inversion must happen after the recursion. `DebugQuery` flattens the hierarchy into a list of (parent, child) index pairs for drawing the skeleton as a wireframe; it is diagnostic only.
