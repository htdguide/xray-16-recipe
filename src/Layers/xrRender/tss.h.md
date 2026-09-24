# src/Layers/xrRender/tss.h

> The recording front end a material pass writes its state through, including the argument-shape rules of the old fixed-function texture combiner.

**Needs** — [`tss_def.h`](tss_def.h.md) · [`tss_def.cpp`](tss_def.cpp.md) · [`Blender_Recorder.h`](Blender_Recorder.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`Blender_Recorder.cpp`](Blender_Recorder.cpp.md) · [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`ResourceManager.cpp`](ResourceManager.cpp.md) · [`ResourceManager_Reset.cpp`](ResourceManager_Reset.cpp.md) · [`ResourceManager_Resources.cpp`](ResourceManager_Resources.cpp.md) · [`ResourceManager_Scripting.cpp`](ResourceManager_Scripting.cpp.md) · [`tss_def.cpp`](tss_def.cpp.md) · [`tss_def.h`](tss_def.h.md) · [`Blender_Recorder_GL.cpp`](../xrRenderGL/Blender_Recorder_GL.cpp.md)
**Tier floor** — T2: a thin recording facade; the tier is fixed only by the numeric state vocabulary it speaks.

## Purpose

Material passes are *recorded*, not configured: the material system replays a pass description and every state it mentions is captured into a [`StateList`](tss_def.h.md). This header is the surface that recording goes through. It is all inline forwarding except for one thing — the texture-combiner helpers, which encode a rule the old graphics API had and the shipped material data depends on.

## Extended state names

The old state vocabulary did not cover everything a modern device needs, so a few names are added above its numeric range:

```text
sampler:  anisotropic_filter, comparison_filter, comparison_function, minimum_lod
render:   alpha_to_coverage
```

They are given values well clear of the original vocabulary's so the two cannot collide, and they are recognised by name in the translation in [`tss_def.cpp`](tss_def.cpp.md). A material file may set them exactly as it sets an original state.

## `set_colour_operation` and `set_alpha_operation`

**Contract** — Record a texture-stage combiner operation together with the arguments it consumes — and record *only* the arguments it consumes.

```text
FUNCTION set_operation(stage, arg1, operation, arg2)
  record the operation
  CASE operation OF
    disable       : record nothing else
    select_first  : record arg1 only
    select_second : record arg2 only
    anything else : record both
```

**Invariants** — This is the rule the file exists for. The old API's combiner reads only the arguments its operation needs, and setting an argument the operation ignores was not merely wasteful: some drivers validated the ignored argument anyway and rejected legal states. The shipped materials were authored against that behaviour, so the pruning must be reproduced — a recorded list with an extra argument is not the same list, and [`equal`](tss_def.cpp.md) will treat two passes that should match as different.

A three-argument form exists for the operations that take a third input; it records the pair as above and then adds the third unconditionally, because no operation that takes a third argument ignores it.

## `set_render_state`

**Contract** — Record a render state. A bare forward; the assertion that the state number fit in the old vocabulary's range was removed, because the extended names above deliberately sit outside it.

## `Recorder`

**Contract** — The object a pass records through: holds one state list, and exposes a named method per category — texture-stage state, sampler state, the colour and alpha combiner pairs and their three-argument forms, and render state — plus `invalidate` to start a fresh recording and an accessor to hand the finished list to the backend.

**Notes** — The split into a combiner helper, a render-state helper and a container that owns both is pure structure; a rebuild can flatten it into one object without losing anything. What must survive is the argument-pruning rule and the extended state names.
