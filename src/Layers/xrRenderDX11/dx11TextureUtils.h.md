# src/Layers/xrRenderDX11/dx11TextureUtils.h

> Declares the two-way pixel-format translation and the one format identifier the engine's vocabulary was missing.

**Needs** — [`dx11TextureUtils.cpp`](dx11TextureUtils.cpp.md)
**Used by** — [`dx11HW.cpp`](dx11HW.cpp.md) · [`dx11SH_RT.cpp`](dx11SH_RT.cpp.md) · [`dx11Texture.cpp`](dx11Texture.cpp.md) · [`dx11TextureUtils.cpp`](dx11TextureUtils.cpp.md)
**Tier floor** — T1: pixel layout names.

## Purpose

Declares the conversions implemented in [`dx11TextureUtils.cpp`](dx11TextureUtils.cpp.md), plus a **synthetic format identifier** for a 32-bit float depth with 8-bit stencil layout. The engine's legacy format enumeration has no name for it, so one is fabricated in the same four-character-tag space the enumeration uses. A rebuild with its own format enumeration simply adds the case and deletes this.

## Exported units

- `ConvertTextureFormat` — engine format to device format, and device format to engine format. Same name, distinguished by argument.
- the synthetic depth-stencil format identifier.
