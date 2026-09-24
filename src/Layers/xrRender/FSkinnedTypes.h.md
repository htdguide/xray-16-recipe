# src/Layers/xrRender/FSkinnedTypes.h

> The eight skinned vertex layouts — one to four bone influences, each in a quantized and a full-precision form — and the packing that hides bone indices and weights inside the tangent frame's unused fourth channels.

**Needs** — [`Include/xrRender/Kinematics.h`](../../Include/xrRender/Kinematics.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`FSkinned.cpp`](FSkinned.cpp.md) · [`FSkinned.h`](FSkinned.h.md)
**Tier floor** — T1: eight exact byte layouts handed to a graphics driver, with hand-quantized fields.

## Purpose

A skinned vertex has to carry a position, a tangent frame, a texture coordinate, and — the part that makes it skinned — up to four bone indices and up to three weights. That is a lot of channels, and the whole design of this file is **fitting them into as few bytes as possible by hiding the skinning data in channels the tangent frame does not use.**

The result is 24 to 28 bytes per vertex regardless of how many bones influence it. That is the decision to carry across; the specific packing is frozen by nothing (the vertices are built at load time from the model file's own format) but it is load-bearing against the shipped *shaders*, which unpack it.

## The quantization

```text
position : signed 16-bit, 32767 units = 12 metres
           -> range ±12 m, resolution about 0.4 mm
texcoord : signed 16-bit, 32767 units = 16
           -> range ±16 texture repeats, resolution about 1/2000 of a repeat
normal,
tangent,
binormal : unsigned 8-bit per component, (v + 1) * 127.5
           -> range -1..+1, about 140 distinct directions per axis
weight   : unsigned 8-bit, v * 255
bone     : unsigned 8-bit or signed 16-bit, stored PRE-MULTIPLIED BY THREE
```

**Invariants**

- **Twelve metres is the hard limit on a skinned model's extent from its own origin.** No creature or weapon in the game approaches it. A rebuild importing larger models must either raise the constant or fall back to the full-precision layout.
- **Bone indices are stored multiplied by three, and divided by three on read.** This is not an encoding quirk: the vertex program indexes a constant array in which each bone occupies *three* four-component registers (a 4×3 transform), so the pre-multiplied value is the register index and the program does no arithmetic. The cost is that the 8-bit index fields cap a model at 85 bones, and the 16-bit ones at far more. Every accessor in this file divides by three to recover the bone number, which is the only place the convention is visible from outside.
- **The position's fourth component is always one.** It is not used. It exists because the position field is a four-component type — the device offers signed 16-bit in pairs and quads, not triples — so the fourth slot is free and wasted. A rebuild on an API with a three-component 16-bit type saves two bytes per vertex.
- The quantized and full-precision forms differ **only** in the position and texture coordinate types. The tangent frame and the skinning data are 8-bit in both. That is deliberate: the precision problems the full-precision form exists to solve are positional, never directional.

## The four packings

```text
ONE BONE      (24 bytes)
  position[4]  : quantized
  normal.xyz + BONE INDEX in the fourth byte
  tangent.xyz  + unused
  binormal.xyz + unused
  texcoord[2]

TWO BONES     (28 bytes)
  position[4]
  normal.xyz   + WEIGHT 0
  tangent.xyz  + unused
  binormal.xyz + unused
  texcoord[2] + BONE INDEX 0, BONE INDEX 1        # four 16-bit slots

THREE BONES   (28 bytes)
  position[4]
  normal.xyz   + WEIGHT 0
  tangent.xyz  + WEIGHT 1
  binormal.xyz + BONE INDEX 2
  texcoord[2] + BONE INDEX 0, BONE INDEX 1

FOUR BONES    (28 bytes)
  position[4]
  normal.xyz   + WEIGHT 0
  tangent.xyz  + WEIGHT 1
  binormal.xyz + WEIGHT 2
  texcoord[2]
  four bone indices packed into one 32-bit value
```

**Invariants**

- **The last weight is never stored.** It is one minus the sum of the others, computed in the vertex program and in every reader here. That is what makes four bones fit: three weights and four indices, not four of each.
- **The two- and three-bone forms widen the texture coordinate field to four components** to carry two bone indices as 16-bit values, while the four-bone form keeps it at two components and packs four indices into a separate 8-bit-per-channel field. The inconsistency is a consequence of the packing being solved separately for each case rather than uniformly, and it is why every accessor is per-layout rather than shared.
- The one-bone layout still carries a full tangent frame with two unused fourth channels. Nothing was found to put there.
- The three-bone form's bone accessor reaches into a *different field* depending on which of the three is asked for. The indices are not in one place; they are wherever there was room.

## Reading a vertex back

**Contract** — every layout can report its position, its bone numbers, its weights, and its *skinned* position: the position transformed by each bone's current render transform and blended by the weights.

```text
FUNCTION skinned_position(vertex, skeleton) -> point
  FOR EACH influence i
    p[i] = bone_transform(skeleton, vertex.bone(i)) applied to vertex.position
  RETURN the weighted sum, with the last weight = 1 - sum of the others
```

**Invariants**

- These read-back paths exist for the **decal and picking systems**, which need a skinned model's triangles in world space on the processor: to cut a wallmark into a creature's skin, and to answer "which bone did this bullet hit". That is why the skinned vertex buffer is created readable, which costs system memory for every skinned model in the scene. See [`FSkinned.cpp`](FSkinned.cpp.md).
- The blend is a weighted *sum* of transformed positions, not a blend of transforms. For two bones it is written as an interpolation, for three and four as a sum. They are the same operation.

## The direction-quantization error check

**Contract** — in a checked build, a helper reports the cosine between an original direction and its quantized form.

**Notes** — It is never called. It is diagnostic scaffolding left in place, and in a shipping build it is a function returning zero. Nothing depends on it.
