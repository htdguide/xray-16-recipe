# src/xrCore/PostProcess — the authored screen effect

Part of chapter 6, [`src/xrCore`](../README.md). Four files describing a small animated
parameter block — and, deliberately, nothing about how it is drawn.

## What this module is responsible for

A *screen effect* is the set of full-frame adjustments the game pushes onto a rendered
image: colour grading, added noise, a blur, a tint, a fade to black. The game raises them for
radiation, a concussion, a drug, a cutscene fade, a hit.

This directory owns two things and no more: **what parameters exist**, and **how two
simultaneous effects combine**. The values are authored offline as keyframed curves, stored
in a frozen binary form, and evaluated here into one parameter block per frame. What the
renderer then *does* with that block belongs to chapter 18.

Keeping the definition in the core, below the renderer, is what lets the game raise an
effect without depending on which backend is loaded.

## Where it sits

It rests on the chunked container ([`../FS.h`](../FS.h.md)) for loading and on the animation
curve machinery next door ([`../Animation/Envelope.hpp`](../Animation/Envelope.hpp.md)) for
evaluation. Chapters 13 and 23 raise effects; chapter 18 consumes the block.

## The load-bearing ideas

**Eleven tracks, one clock.** An effect is eleven independent keyframed curves — colour
components, noise parameters, blur, duplication — driven by one playback time. They are
evaluated together and yield one block.

**Combination is per-parameter, and the rules differ.** This is the decision a rebuild must
get right, because it is what makes two overlapping effects look correct rather than
saturated: **scalars add**, **noise takes the stronger of the two**, and **colour-mapping
textures form a two-slot crossfade** rather than blending. A single uniform rule for all
eleven produces visibly wrong results the moment two effects overlap.
[`PPInfo.cpp`](PPInfo.cpp.md) is that rule set.

**The binary form is frozen.** The shipped game data contains authored effects, so the
loader must read the layout as written.

## The twins

| File | Role |
|---|---|
| [`PPInfo.hpp`](PPInfo.hpp.md) | **The complete parameter set** an effect can push onto a frame, and the combination rules. Substantive. |
| [`PPInfo.cpp`](PPInfo.cpp.md) | **How two parameter blocks combine**: scalars add, noise takes the stronger, colour maps crossfade in two slots. |
| [`PostProcess.hpp`](PostProcess.hpp.md) | Declares the animated effect: eleven keyframed tracks on one playback clock. |
| [`PostProcess.cpp`](PostProcess.cpp.md) | Loading an authored effect from its frozen binary form and evaluating its eleven curves at a time. |
