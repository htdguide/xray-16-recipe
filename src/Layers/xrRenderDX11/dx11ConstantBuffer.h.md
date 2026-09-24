# src/Layers/xrRenderDX11/dx11ConstantBuffer.h

> Declares the shader constant block: its typed setters, its direct-write escape hatch and its identity test.

**Needs** — [`dx11ConstantBuffer.cpp`](dx11ConstantBuffer.cpp.md) · [`dx11ConstantBuffer_impl.h`](dx11ConstantBuffer_impl.h.md) · [`xrRender/r_constants.h`](../xrRender/r_constants.h.md)
**Used by** — [`dx11ConstantBuffer.cpp`](dx11ConstantBuffer.cpp.md) · [`dx11ConstantBuffer_impl.h`](dx11ConstantBuffer_impl.h.md) · [`dx11ResourceManager_Resources.cpp`](dx11ResourceManager_Resources.cpp.md) · [`dx11r_constants.cpp`](dx11r_constants.cpp.md) · [`dx11r_constants_cache.cpp`](dx11r_constants_cache.cpp.md) · [`dx11r_constants_cache.h`](dx11r_constants_cache.h.md)
**Tier floor** — T1: a reference-counted named resource wrapping a device buffer and a raw byte image.

## Purpose

Declares the surface implemented in [`dx11ConstantBuffer.cpp`](dx11ConstantBuffer.cpp.md) and [`dx11ConstantBuffer_impl.h`](dx11ConstantBuffer_impl.h.md).

## Exported units

- **the constant block** — constructed from a compiled program's reflection of one block; `Similar` for deduplication; `GetBuffer` to hand the device handle to the binder; `Flush` to upload.
- **typed writes** — `set` for a matrix, a four-component vector, a scalar float and a scalar integer; `seta` for the same into an indexed element of an array member. Each takes the constant record (which carries the declared type) and the *load* record (which carries the byte offset and the shape), so the value's declared shape is checked against what the caller is writing.
- **`AccessDirect`** — returns a raw pointer into the host image for a caller that wants to write a block of bytes itself, or nothing when the requested span would overflow the block. The caller is trusted to write only within the span it asked for.
- **the reference type** — blocks are reference-counted and named, so that a deduplicated block outlives any one program.

**Notes** — The block is explicitly non-copyable because it owns a device handle and a host allocation; in a rebuild this is simply a resource with move-only ownership.
