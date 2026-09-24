# src/Layers/xrRenderDX11/dx11BufferUtils.cpp

> Buffer creation for this backend: the two lifetimes a buffer can have, and the translation of the engine's frozen vertex-layout description into something the device will accept.

**Needs** — [`xrRender/BufferUtils.h`](../xrRender/BufferUtils.h.md) · [`CommonTypes.h`](CommonTypes.h.md) · [`dx11HW.h`](dx11HW.h.md) · `xrCore/FlexibleVertexFormat.h` · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx11ConstantBuffer.cpp`](dx11ConstantBuffer.cpp.md) · [`dx11ResourceManager_Resources.cpp`](dx11ResourceManager_Resources.cpp.md)
**Tier floor** — T1: byte strides, explicit buffer usage classes, and a mapped pointer handed to the caller.

## Purpose

Chapter 18 declares four buffer shapes and this file fills them in. The four are not arbitrary; they are the two *lifetimes* crossed with the two *roles*:

- **Staging buffers** (vertex and index) hold geometry that is built once and then never changes — level geometry, model meshes. The data is assembled in host memory and uploaded in one shot as an immutable device buffer, after which the host copy is thrown away unless the caller asked to keep reading it.
- **Stream buffers** (vertex and index) are the per-frame scratch — particles, interface quads, debug lines, decals. They are device buffers the renderer writes into directly, in a ring discipline.

The second job here is the **vertex-layout translation**, which is the one piece of this file a rebuild on any API has to reproduce exactly, because the input side of it is frozen in the game's model files.

## State

```text
RECORD StagingBuffer            # vertex or index
  size            : int
  allow_read_back : bool        # keep the host copy after upload
  host            : bytes       # released at upload unless read-back was requested
  device          : handle      # created at upload; absent until then

RECORD StreamBuffer             # vertex or index
  device : handle               # created dynamic and host-writable at construction
```

Invariant on a staging buffer: it may be uploaded **once**. A second upload is a programming error, because the device buffer it creates is immutable. `IsValid` means "has been uploaded".

## `VertexStagingBuffer` / `IndexStagingBuffer`

**Contract** — `Create(size, allow_read_back)` allocates host memory only. `Map(offset, size, read)` hands back a pointer into that host memory — no device involvement, so it never blocks and can be done on any thread, including a level-loading worker. `Unmap(flush)` does nothing unless `flush` is set, in which case it creates the immutable device buffer from the host image, registers its size with the memory statistics, and frees the host copy unless read-back was requested. `Destroy` releases both.

```text
FUNCTION upload(buffer)
  REQUIRE buffer.device is absent        # one-shot: immutable buffers cannot be refilled
  buffer.device = device.create_buffer(role, immutable, buffer.host, buffer.size)
  account for buffer.size in the video-memory statistics
  IF NOT buffer.allow_read_back THEN release buffer.host
```

**Invariants** — Mapping a staging buffer for *reading* requires that it was created with read-back, because otherwise the host copy no longer exists. The renderer needs read-back for meshes it re-processes on the host — collision proxies, progressive geometry.

**Notes** — This is the answer to the seam's "what may be created off the render thread" question, and it is the reason the split exists at all: **assembling geometry touches no device state**, so level loading fills host buffers on worker threads and only the final upload is serialized. A rebuild on an API with explicit transfer queues can make that upload asynchronous too; a rebuild that maps a device buffer during loading will stall the frame loop.

Several statistics hooks account buffer sizes into a per-category video-memory total. That total is player-visible in the debug overlay and is the only way the engine knows how much device memory it is using.

## `VertexStreamBuffer` / `IndexStreamBuffer`

**Contract** — `Create(size)` allocates a dynamic, host-writable device buffer of the given size immediately. `Map(offset, size, flush)` maps it and returns a pointer offset into it. `Unmap` releases it. The `flush` flag chooses between the two map disciplines, and that choice is the whole design:

```text
discard      when the writer has wrapped to the start of the buffer:
             "the previous contents are dead, give me fresh storage"
no-overwrite otherwise:
             "I am writing a region no pending draw reads; do not synchronize"
```

**Invariants** — The caller — chapter 18's geometry streams — owns the ring discipline: it advances an offset, and when the next allocation will not fit it wraps and asks for a discard. Getting this wrong in either direction is invisible until it is not: discarding too often throws away the frame's earlier vertices, and never discarding overwrites data the device is still reading.

**Notes** — The source flags two limitations. The map is always issued on the immediate submission context, so a worker thread recording into a deferred context cannot use these buffers safely; and the mapping belongs in the command list rather than in the buffer, for the same reason. A rebuild should give each context its own stream buffers.

## `ConvertVertexDeclaration`

**Contract** — translate the engine's vertex declaration — the frozen, legacy-format element list that ships inside model files — into the device's input-element description list. One entry per element plus a terminator; the declaration's own terminator entry is not translated.

```text
FOR EACH element IN declaration (excluding the terminator)
  out.semantic_name  = name_for(element.usage)      # POSITION, NORMAL, TEXCOORD, ...
  out.semantic_index = element.usage_index
  out.format         = format_for(element.type)
  out.slot           = element.stream
  out.byte_offset    = element.offset
  out.per_vertex, no instancing step
```

**Invariants** — Every element is per-vertex; **this engine never uses per-instance vertex data**, so an input layout has no instancing step anywhere. Byte offsets come straight from the declaration, which means the model file's own packing is authoritative and must not be re-derived.

## `ConvertVertexFormat` / `ConvertSemantic`

**Contract** — map one element type or one usage to the device's equivalent, failing loudly on anything unmapped.

**Notes** — Three details of the type mapping are load-bearing for a port:

- The packed-colour type maps onto an eight-bit normalized four-channel format whose channel order is *not* the same; the source notes the shader must swizzle the components back. This is exactly the kind of thing that makes a port's first frame look wrong in a way that is hard to trace.
- The integer types (four unsigned bytes, two or four signed shorts) arrive in the shader as *integers*, whereas the previous generation delivered them as floats in the same numeric range. The shipped shaders were written for the float behaviour, so a shader reading a bone index or a packed normal must convert. A rebuild must decide this once and apply it everywhere.
- Two packed three-component types the older format allowed have no equivalent and are simply unavailable; no shipped model uses them.

The usage mapping covers position, blend weight, blend index, normal, point size, texture coordinate, tangent, binormal, transformed position and colour. Tessellation factor, fog, depth and sample usages are not mapped — nothing in the game data declares them.

## `GetFVFVertexSize` / `GetDeclVertexSize` / `GetDeclLength`

**Contract** — stride and length queries that delegate to the shared, backend-independent vertex-format helper. They exist here only because the declaration type is backend-spelled; the arithmetic is the same everywhere.

## `CreateConstantBuffer`

**Contract** — create a dynamic, host-writable buffer of a given size for shader constants. Exposed through the shared buffer interface because [`dx11ConstantBuffer.cpp`](dx11ConstantBuffer.cpp.md) needs it and should not reach into the device itself.
