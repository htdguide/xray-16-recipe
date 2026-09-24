# src/Layers/xrRenderGL/glTextureUtils.h

> Declares the format lookup: one engine format name to one device internal format.

**Needs** — [`glTextureUtils.cpp`](glTextureUtils.cpp.md)
**Used by** — [`glSH_RT.cpp`](glSH_RT.cpp.md) · [`glTextureUtils.cpp`](glTextureUtils.cpp.md)
**Tier floor** — T1: it names byte layouts a driver allocates storage from.

## Purpose

Declares the single converter implemented in [`glTextureUtils.cpp`](glTextureUtils.cpp.md): `convert_texture_format(engine_format) -> device_internal_format`. It covers only the formats the *engine itself* asks for when it creates render targets and procedural textures. Formats that arrive from shipped image files never pass through here — those come with their own descriptor from the image codec (see [`glTexture.cpp`](glTexture.cpp.md)).
