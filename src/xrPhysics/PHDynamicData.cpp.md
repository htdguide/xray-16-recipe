# src/xrPhysics/PHDynamicData.cpp

> Nothing. The file's entire content is disabled.

**Needs** — [`PHDynamicData.h`](PHDynamicData.h.md)
**Used by** — [`PHDynamicData.h`](PHDynamicData.h.md)
**Tier floor** — T4: it contributes no behaviour.

## Purpose

Stateless, and — as shipped — behaviourless. Every definition in this file sits inside a disabled
block; the translation unit compiles to nothing. The live part of the pair is entirely in
[`PHDynamicData.h`](PHDynamicData.h.md), which is where the matrix conversions the module actually
uses live.

## What it would have been

The disabled code implements a tree of simulated bodies, each holding its own interpolation and a
recorded rest pose, with three operations: compute every descendant's transform relative to its
parent in one recursive walk; push every descendant's interpolation sample in one walk; and
interpolate one node's transform either in world space or relative to its parent.

The intent is a per-frame cache of a whole ragdoll's pose, computed once, replacing per-bone
computation on demand.

**Notes** — a rebuild should not port it. It is recorded here because a reader comparing the recipe
to the original will find a 200-line file and wonder what happened to it, and because the problem
it addresses is real: a ragdoll with thirty bones currently recomputes parent-relative transforms
per bone, per frame, through a callback. If that shows up in a profile, this file is the sketch of
the intended fix, and the reason it was abandoned is not recorded anywhere in the source.
