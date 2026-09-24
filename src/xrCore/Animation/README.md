# src/xrCore/Animation — the skeletal animation data model

Part of chapter 6, [`src/xrCore`](../README.md). Eleven files that define what a skeleton
is, what a motion is, and how both are stored — and nothing at all about how either is
played, blended or drawn.

## What this module is responsible for

Two data models live here, and keeping them apart is the single most useful thing a reader
can do before opening any of the twins.

The **authored** model is what a modelling tool exports: a bone hierarchy with named
parents, joint limits in the author's units, collision shapes, and motions expressed as six
independent *curves* over time — three for position, three for orientation — each with
per-key interpolation shapes and out-of-range behaviour. It is expressive, floating-point,
and expensive to evaluate.

The **runtime** model is what the engine actually plays: the same motions resampled onto a
fixed grid and quantized to integers, laid out so that an animation bank can be pointed at
a mapped file region and used without parsing. It is cheap, lossy, and frozen.

The module owns the conversion between them, both serialization formats across every
version the shipped data uses, and the sharing scheme that lets a hundred instances of one
creature reference one copy of its animation bank. It owns no playback: blending, the
animation state machine, inverse kinematics and skinning all live in the renderer and the
game.

## Where it sits

It rests on the rest of chapter 6 — the chunked container ([`../FS.cpp`](../FS.cpp.md)) for
every read and write, the string interner for bone and motion names, the blob interner
([`../xrsharedmem.cpp`](../xrsharedmem.cpp.md)) for the key arrays — and on chapter 3's
vector and quaternion types. Nothing above chapter 6 is referenced.

Chapter 18 onward is the consumer: the renderer's skinned-mesh path reads these records and
the game's animation layer drives them.

## The load-bearing ideas

**Six curves, not a pose stream.** An authored motion is not a sequence of poses; it is six
scalar functions of time, keyed independently. Two channels of one bone can therefore have
keys at different times and different interpolation shapes. Everything about
[`Envelope.hpp`](Envelope.hpp.md) and [`interp.cpp`](interp.cpp.md) follows from this, and
so does the fact that converting to the runtime form requires *resampling*, not copying.

**The runtime form is quantized, and the quantizer is the format.** Positions and rotations
are stored as integers against fixed scales, sampled at a fixed rate. The five constants
that define this are in [`SkeletonMotionDefs.hpp`](SkeletonMotionDefs.hpp.md) and are the
first thing to read in this directory: they decide the precision of every animation in the
game, and changing one changes the meaning of every byte in every shipped animation bank.

**Animation banks are shared, not copied.** A bank is loaded once per model file and
referenced by every instance. The keys are not copied out of the mapped file at all where
the layout permits. This is why the loading path in
[`SkeletonMotions.cpp`](SkeletonMotions.cpp.md) reads as an exercise in *not* doing things.

**A bone's joint limits mean opposite things on the two sides.** The authoring tool and the
physics solver disagree about the sign of a rotation limit, and the serializer flips it.
[`Bone.cpp`](Bone.cpp.md) records exactly where; a rebuild that misses it produces ragdolls
that fold the wrong way.

**Partitions are how a body plays two animations at once.** A skeleton is divided into named
bone subsets, and a motion is played into a partition rather than onto the whole skeleton.
That is what lets an upper body reload while the legs walk. The partition tables travel with
the bank.

## The twins

| File | Role |
|---|---|
| [`SkeletonMotionDefs.hpp`](SkeletonMotionDefs.hpp.md) | **The five constants the runtime format is built on** — sample rate, quantizer scales, partition count. Read first. |
| [`SkeletonMotions.hpp`](SkeletonMotions.hpp.md) | **The runtime animation format**: quantized key arrays, the shared bank, the partition and motion-definition tables that drive blending. Substantive. |
| [`SkeletonMotions.cpp`](SkeletonMotions.cpp.md) | Loading a bank out of a model file without copying a key, and sharing what it can. |
| [`Bone.hpp`](Bone.hpp.md) | **The skeleton's data model**: a bone, its joint, its collision shape, and how a skinned vertex names its bones. Substantive. |
| [`Bone.cpp`](Bone.cpp.md) | Bone serialization: which chunks carry what, which are optional, and the joint-limit sign flip. |
| [`BoneEditor.cpp`](BoneEditor.cpp.md) | Authoring operations on a bone — and the one the engine itself needs: clamping a pose to what the joint allows. |
| [`Motion.hpp`](Motion.hpp.md) | **The authored animation model**: six curves per object or per bone, plus the playback parameters that decide how a motion blends. Substantive. |
| [`Motion.cpp`](Motion.cpp.md) | Motion serialization across five format versions, and reconciling a motion's bone list with a skeleton's. |
| [`Envelope.hpp`](Envelope.hpp.md) | **One animation curve**: keyed, interpolated, with a shape per key and a behaviour outside the keyed range. Substantive. |
| [`Envelope.cpp`](Envelope.cpp.md) | Curve editing: find, insert, delete, retime, simplify. |
| [`interp.cpp`](interp.cpp.md) | **Curve evaluation**: the six key shapes, the six out-of-range behaviours, and the tangent rules that make them agree. Substantive. |
