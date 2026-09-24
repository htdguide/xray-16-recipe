# src/Layers/xrRender/dxParticleCustom.cpp

> Nothing. A compilation anchor for the type declared in the header.

**Needs** — [`dxParticleCustom.h`](dxParticleCustom.h.md) · [`FBasicVisual.h`](FBasicVisual.h.md)
**Used by** — [`dxParticleCustom.h`](dxParticleCustom.h.md)
**Tier floor** — T4: it exists to give a build system a file to compile.

## Purpose

`Stateless.` The file contains no definitions. It exists so that the type declared in [`dxParticleCustom.h`](dxParticleCustom.h.md) has exactly one translation unit in which its compiler-generated constructor, destructor and method table are emitted, rather than one per including file.

**In a rebuild this file does not exist.** It is an artifact of a compilation model that emits a type's implicit members wherever the type is used and needs one place designated as the owner. Any language that compiles a type once deletes it.
