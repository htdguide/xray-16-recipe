# src/Layers/xrRender/Utils/dxHashHelper.h

> Declares the state-description hasher, with the per-byte fold inline.

**Needs** — _(none)_
**Used by** — [`dx11StateUtils.cpp`](../../xrRenderDX11/dx11StateUtils.cpp.md)
**Tier floor** — T2: a declaration only.

## Purpose

Declares the hasher implemented in [`dxHashHelper.cpp`](dxHashHelper.cpp.md). The per-byte fold itself lives here rather than in the implementation, because it is called once per byte of every state description built during level load and the call overhead would dominate it — a packaging decision with no bearing on a rebuild.

## Exported units

- **`dxHashHelper`** — construct to start a hash, `AddData` to fold in a run of bytes, `GetHash` to read the finished value. The lookup table is shared across all instances and built on first use.
