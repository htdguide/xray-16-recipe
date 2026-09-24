# src/Layers/xrRender/Blender.h

> Declares the material-template base class and its identity record.

**Needs** — [`Blender_Recorder.h`](Blender_Recorder.h.md) · [`xrEngine/Properties.h`](../../xrEngine/Properties.h.md)
**Used by** — [`Blender.cpp`](Blender.cpp.md) · [`Blender_Recorder.cpp`](Blender_Recorder.cpp.md) · [`Blender_Recorder_R2.cpp`](Blender_Recorder_R2.cpp.md) · [`ResourceManager.cpp`](ResourceManager.cpp.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`ResourceManager_Loader.cpp`](ResourceManager_Loader.cpp.md) · [`ResourceManager_Scripting.cpp`](ResourceManager_Scripting.cpp.md) · [`BlenderDefault.cpp`](blenders/BlenderDefault.cpp.md) · [`BlenderDefault.h`](blenders/BlenderDefault.h.md) · [`Blender_Blur.cpp`](blenders/Blender_Blur.cpp.md) · [`Blender_Blur.h`](blenders/Blender_Blur.h.md) · [`Blender_BmmD.cpp`](blenders/Blender_BmmD.cpp.md) · [`Blender_BmmD.h`](blenders/Blender_BmmD.h.md) · [`Blender_BmmD_deferred.cpp`](blenders/Blender_BmmD_deferred.cpp.md) · _and 79 more_
**Tier floor** — T2: a declaration only.

## Purpose

Declares the surface implemented in [`Blender.cpp`](Blender.cpp.md) and inherited by every concrete template in `blenders/`. It also fixes the *binary* layout of the identity record, which is part of the shipped material library's format.

## Exported units

- **`CBlender_DESC`** — the identity record written into the material library: class identifier, name, authoring machine, authoring time, parameter-block version. One method, `Setup`, stamps a fresh one.
- **`IBlender`** — the material template base: the shared parameters (priority, strict back-to-front, base texture name, base transform name), the three capability queries, the save/load pair for the parameter block, the `Compile` entry point, and the static create/destroy pair that routes through the active backend.

The class derives from the authoring tools' property-bag base, which is what gives its parameters an editable, serializable description without a per-subclass schema. That inheritance is incidental to running the game and load-bearing to writing the library.
