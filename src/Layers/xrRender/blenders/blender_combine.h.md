# src/Layers/xrRender/blenders/blender_combine.h

> Declares the deferred-resolve templates.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`blender_combine.cpp`](blender_combine.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the templates implemented in [`blender_combine.cpp`](blender_combine.cpp.md). Both answer no to both capability queries and carry no parameters; their identity is the zero class tag, so they are constructed by the render target and never looked up from the material library.

## Exported units

- **`CBlender_combine`** — six elements: the deferred resolve, then the four antialias-by-distortion composite variants, then one reserved and empty.
- **`CBlender_combine_msaa`** — the same six against a multisampled frame. Declared only in the generations that support multisampling. Constructed with a name and a definition string; when the name is present the definition is parsed as the sample index and pushed into renderer state for the duration of the compile, so that the shader compiler picks it up as a macro.
