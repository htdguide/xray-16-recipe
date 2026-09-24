# src/Layers/xrRender/BufferUtils.h

> Vertex and index buffer ownership: four buffer kinds distinguished by whether the host keeps a copy and whether writes append or discard, plus the vertex-layout description helpers.

**Needs** — [`R_Backend.h`](R_Backend.h.md) · [`FVF.h`](FVF.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`D3DUtils.h`](D3DUtils.h.md) · [`DetailManager.h`](DetailManager.h.md) · [`DetailManager_VS.cpp`](DetailManager_VS.cpp.md) · [`FBasicVisual.h`](FBasicVisual.h.md) · [`FSkinned.cpp`](FSkinned.cpp.md) · [`FVisual.cpp`](FVisual.cpp.md) · [`R_Backend.cpp`](R_Backend.cpp.md) · [`R_Backend.h`](R_Backend.h.md) · [`R_DStreams.cpp`](R_DStreams.cpp.md) · [`R_DStreams.h`](R_DStreams.h.md) · [`dx113DFluidGrid.cpp`](../xrRenderDX11/3DFluid/dx113DFluidGrid.cpp.md) · [`dx113DFluidRenderer.cpp`](../xrRenderDX11/3DFluid/dx113DFluidRenderer.cpp.md) · [`dx11BufferUtils.cpp`](../xrRenderDX11/dx11BufferUtils.cpp.md) · [`dx11ConstantBuffer.cpp`](../xrRenderDX11/dx11ConstantBuffer.cpp.md) · _and 2 more_
**Tier floor** — T1: every method is a byte-offset, byte-size contract against driver memory, and the map/unmap pair hands out a raw writable region of it.

## Purpose

Everything the renderer draws comes out of a vertex buffer and an index buffer, and there are exactly two ways the engine uses one: *fill it once and forget it* (level geometry, model meshes) or *refill part of it every frame* (particles, decals, the user interface, skinned output). This file names four classes for those two cases crossed with the vertex/index distinction, so that the wrong usage is a type error rather than a driver stall.

The split is not arbitrary and a rebuild should keep it. Mapping a static buffer every frame and mapping a dynamic buffer with the wrong discard hint are the two classic ways to lose a renderer's performance to synchronization, and neither is visible at the call site unless the buffer's *kind* carries the intent.

## Vertex layout description

```text
FUNCTION fvf_vertex_size(format_bits) -> int
FUNCTION decl_vertex_size(elements, stream_index) -> int
FUNCTION decl_length(elements) -> int
FUNCTION decl_equal(a, b) -> bool
```

**Contract** — compute the byte stride of a vertex from either of the two ways the engine describes one, and compare two descriptions for identity.

**Invariants**

- Two vertex descriptions are equal when they are the same length and byte-identical. This is deliberately a *byte* comparison, not a semantic one: the descriptions come from shipped model files and are used as a cache key for compiled vertex-input state. Two descriptions that mean the same thing but differ in a padding byte must miss the cache rather than risk a mismatch with the shader they were compiled against.
- The element list is terminated by a sentinel element, so the length is discovered by scanning rather than stored. Any rebuild is free to carry an explicit count; the terminator is a format artefact of the graphics API the original was written against.

**Notes** — Two description styles coexist because the shipped model data uses the older *packed bit-field* style (a single integer naming which attributes are present, in a fixed order) and the renderer's own geometry uses the newer *element list* style (semantic, index, type, stream, offset). [`FVF.h`](FVF.h.md) holds the bit-field vocabulary. A rebuild must read the packed style, because it is in the model files, and should convert to the element style on load rather than carrying both through the renderer.

## Staging buffers — `VertexStagingBuffer`, `IndexStagingBuffer`

**Contract** — a device buffer with an intermediate host-side buffer used to fill it. Created with a size and a *read-back* policy. Mapping returns a writable host region; unmapping uploads it. Under the push-once policy the host buffer is released at unmap; under the persistent policy it survives and may be mapped again for reading.

```text
FUNCTION create(size, allow_read_back)
FUNCTION map(offset, size, for_read) -> writable region
FUNCTION unmap(flush)
FUNCTION discard_host_buffer()
FUNCTION device_handle() -> buffer handle
```

**Invariants**

- Mapping for read is only legal when the buffer was created with read-back allowed, and such a map yields data that must not be modified. The engine's skinning path is the reason read-back exists at all: it needs the source mesh vertices back after upload.
- The host buffer may be discarded at any time by the owner; after that the buffer is upload-only and a read map is an error. Discarding is the only way to reclaim the host copy of a large static mesh, and the engine does it explicitly rather than guessing.
- A staging buffer is reference counted, and the last release destroys both the host and the device side. The count exists because a single mesh's buffer is shared by every model instance and by every level-of-detail variant of it; the buffer outlives any one of them.

**Notes** — "Push once" versus "persistent" is a *memory* policy, not a mutability policy: both are freely re-uploadable. The distinction is only whether the engine keeps its own copy. A rebuild in a language whose buffers can report their own contents back collapses this to one class with an optional shadow copy.

The index staging buffer additionally takes a *managed* flag, which asks the device to keep its own backing store so the buffer survives a device reset without the engine re-uploading it. On APIs without that concept the flag is ignored and the engine's own reset path refills the buffer; the contract the caller relies on is only "this buffer's contents are still there after a resolution change".

## Stream buffers — `VertexStreamBuffer`, `IndexStreamBuffer`

**Contract** — a write-only device buffer with no host copy, mapped in place and refilled continuously. Mapping in *append* mode (the default) promises not to touch anything previously written, so the driver need not stall; mapping in *flush* mode declares the whole previous contents dead and lets the driver hand back fresh memory.

**Invariants**

- The caller is responsible for never mapping a region in append mode that a still-pending draw is reading. The engine guarantees this by allocating monotonically forward through the buffer and flushing only when it wraps — see [`R_DStreams.cpp`](R_DStreams.cpp.md), which is the only real user.
- Reading from a stream buffer is undefined, not merely slow. Nothing in the engine does it.

**Notes** — The append/flush pair is the classic orphaning idiom for dynamic geometry and is the single most performance-load-bearing decision in this file. Every rebuild on every graphics API needs an equivalent, whatever its local spelling: a ring buffer with explicit fences, a per-frame arena, or the API's own discard hint.

## `create_constant_buffer(handle, size)`

**Contract** — allocates a shader constant buffer of the given byte size. Separate from the vertex and index paths because constant buffers have their own alignment rule and are always dynamic.
