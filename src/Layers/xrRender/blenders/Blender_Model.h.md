# src/Layers/xrRender/blenders/Blender_Model.h

> Declares the default dynamic-model template.

**Needs** — [`Blender.h`](../Blender.h.md) · [`Blender_Recorder.h`](../Blender_Recorder.h.md)
**Used by** — [`Blender_Model.cpp`](Blender_Model.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the template implemented in [`Blender_Model.cpp`](Blender_Model.cpp.md).

## Exported units

- **`CBlender_Model`** — the model class tag at parameter version 2. Inherits the base's "no" to both detail and lightmap capability: a model takes neither. Three parameters: a blend flag, an alpha reference (default 32), and a four-way tessellation selector.
