# src/xrServerEntities/PHNetState.cpp

> Writes and reads a rigid body's state three ways — full precision, quantized against a known box, and a ragdoll's worth of either — and decides which parts of that state are worth sending at all.

**Needs** — [`PHNetState.h`](PHNetState.h.md) · [`xrNetServer/NET_Shared.h`](../xrNetServer/NET_Shared.h.md) · [Data: network protocol](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)
**Used by** — [`PHNetState.h`](PHNetState.h.md)
**Tier floor** — T1: exact field order and exact quantization widths, both frozen against saves and the wire.

## Purpose

A physics object's entity record has to carry enough state that reloading it puts the body
back where it was, awake or asleep, moving as it was. This file is that serialization, and
its content is two judgements: **what is worth storing** and **at what precision**.

## State

```text
RECORD BodySnapshot
  linear_velocity     : vector of real (32-bit)
  angular_velocity    : vector of real (32-bit)   # never serialized; see below
  force               : vector of real (32-bit)   # never serialized
  torque              : vector of real (32-bit)   # never serialized
  position            : vector of real (32-bit)
  previous_position   : vector of real (32-bit)   # never serialized; reconstructed
  orientation         : quaternion of real (32-bit)
  previous_orientation: quaternion of real (32-bit)   # never serialized; reconstructed
  awake               : bool                      # written as one byte

RECORD SkeletonSnapshot
  bone_mask   : int (64-bit)     # which bones participate; all bits set by default
  root_bone   : int (16-bit)
  box_min     : vector of real (32-bit)
  box_max     : vector of real (32-bit)
  bones       : list<BodySnapshot>   # count written as 16-bit
```

**Invariants**

- The bounding box must be non-degenerate: a box whose minimum equals its maximum makes the
  quantization divide by zero, and every entry point asserts against it. The default box is
  ±100 metres on each axis, which is the fallback for a skeleton whose real extent is not
  yet known.
- The bone mask defaults to all bits set — every bone participates unless something clears
  a bit. Its width caps a physics skeleton at 64 bones.
- A loaded snapshot always has its previous position equal to its position and its previous
  orientation equal to its orientation. There is no motion history across a load, so the
  first step after a restore computes zero implied velocity from the deltas.

## `write` (full precision)

**Contract** — writes linear velocity, position, orientation and the awake flag, in that
order, at full 32-bit float precision. Does **not** write angular velocity, force or torque.

**Notes** — the omission is the decision. Force and torque are re-derived every step from
the world (gravity, contacts, whatever is pushing), so storing them stores one frame of
noise. Angular velocity is the interesting omission: it *is* real state, and dropping it
means a spinning barrel stops spinning across a save or a network correction. The original
accepted that, presumably because the visual cost is small and the field is three floats per
body per update on a protocol that is already the bandwidth bottleneck. A rebuild that wants
spinning to survive adds it — and must then version-gate, because the byte layout changes.

## `read` (full precision)

**Contract** — reads the four written fields and **zeroes** angular velocity, force and
torque rather than leaving them. Copies the orientation into the previous orientation. The
load variants additionally copy the position into the previous position.

**Invariants** — zeroing rather than preserving is what makes a load deterministic: whatever
the body's live state was before, after a read it is exactly the stream's content plus
zeros.

## `write` / `read` (quantized against a box)

**Contract** — the compact form used for ragdolls. Position is written as three bytes, each
one component mapped into `0..255` across the corresponding span of the supplied box.
Orientation is written as four bytes, each component mapped across `-1..+1`. The awake flag
is a byte. Linear velocity is **not** written at all in this form.

Reading clamps every component back into its range after dequantizing, because the byte
round-trip can land marginally outside.

```text
FUNCTION write_quantized(out, snapshot, box_min, box_max)
  FOR EACH axis IN x, y, z
    out.write_byte_quantized(snapshot.position[axis], box_min[axis], box_max[axis])
  FOR EACH component IN x, y, z, w
    out.write_byte_quantized(snapshot.orientation[component], -1, +1)
  out.write_byte(snapshot.awake)
```

**Notes**

- **One byte of position across the box.** With the default ±100 metre box that is 0.8
  metres of error per axis, which would be absurd for a free object; it is acceptable here
  because the box is meant to be the *ragdoll's own* bounds, a couple of metres across,
  giving centimetre precision. A caller that forgets to set the real box gets a visibly
  wrong ragdoll, which is why the degenerate-box assertion exists and why the default is
  deliberately a bad box rather than a plausible one.
- **All four quaternion components are written.** The obvious economy — write three, derive
  the fourth from the unit constraint — is present in the source as commented-out code and
  was abandoned, because the sign of the fourth component is lost and recovering it costs
  another bit plus a branch. Four bytes it is.
- **Velocity is dropped entirely in the quantized form.** A ragdoll's bones are re-solved
  from their positions; carrying per-bone velocity would double the message for state the
  solver will overwrite within a step.

## the six entry points

**Contract** — export/import are the network pair, save/load the persistence pair, and they
are *the same bytes*: save delegates to export, load delegates to import. The only
difference is that load also resets the previous position. Two further variants take the
bounding box and use the quantized form.

**Notes** — that the network and save encodings are identical here is worth stating,
because it is *not* true elsewhere in this chapter: most entity records have a genuinely
different update packet from their save record. For a rigid body, the state that matters is
the same in both directions, so the split is nominal. What differs is precision — full for a
single body, quantized for a skeleton — and that choice is made by the caller, not by the
medium.

## skeleton `write` / `read`

**Contract** — writes the bone mask, the root bone, the box, the bone count, then each
bone's snapshot quantized against that box. Reading clears the bone list first and rebuilds
it from the stream, re-establishing the box before any bone is dequantized (the box is
needed to decode them).

**Invariants** — the box is written *before* the bones and read *before* them, because it is
the dequantization basis. Reordering those two makes the stream unreadable.

**Notes** — the source carries an unresolved note that writing a skeleton twice in a row
produces different results. The cause is that writing does not clear the bone list while the
live physics keeps updating it, so the second write sees a later state. It is recorded here
as an open question in the original, not a decision: a rebuild should treat "serialize a
snapshot" as reading a consistent set of bones once, which sidesteps it.

A read entry point taking a plain file stream is declared for the skeleton but never
defined; only the packet form exists. Nothing calls the missing one.
