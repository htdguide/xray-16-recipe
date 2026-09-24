# src/Layers/xrRenderDX11/dx11ConstantBuffer.cpp

> One shader constant block: a host-side shadow of its bytes, the device buffer it is uploaded into, and the layout fingerprint that lets many programs share one instance.

**Needs** — [`dx11ConstantBuffer.h`](dx11ConstantBuffer.h.md) · [`dx11ConstantBuffer_impl.h`](dx11ConstantBuffer_impl.h.md) · [`xrRender/BufferUtils.h`](../xrRender/BufferUtils.h.md) · [`dx11BufferUtils.cpp`](dx11BufferUtils.cpp.md) · [`dx11ResourceManager_Resources.cpp`](dx11ResourceManager_Resources.cpp.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx11ConstantBuffer.h`](dx11ConstantBuffer.h.md)
**Tier floor** — T1: it is a byte image with a device mirror, written through raw offsets computed by the shader compiler.

## Purpose

A compiled shader program declares its uniform inputs grouped into named blocks. This is one such block, reconstructed from the compiler's reflection data. It owns two things: a host-side byte image the renderer writes into through offsets, and the device-side buffer that image is copied to when a draw needs it.

It also answers the question that makes the whole constant system affordable: **are two blocks from two different programs the same block?** If two programs declare a block with the same name, the same member layout and the same member names, the engine keeps *one* instance and both programs' constant tables point at it. So a value written once — the camera transform, the fog parameters — is written once and uploaded once, no matter how many materials read it.

## State

```text
RECORD ConstantBlock
  name          : text          # the block's declared name; part of the identity
  kind          : enum          # the block's declared class (ordinary / tightly-bound)
  member_types  : list<TypeDesc># one per declared member, as the compiler describes it
  member_names  : list<text>    # parallel to member_types
  layout_hash   : int (32-bit)  # checksum over member_types; a cheap pre-filter for identity
  size          : int           # bytes; the device buffer and the shadow are both this big
  shadow        : bytes         # host image, written by the constant setters
  device_buffer : handle
  dirty         : bool          # shadow has been written since the last upload
```

Invariants: `size` is what the compiler reported for the block and never changes; every write is bounds-checked against it in a debug build. `member_types` and `member_names` are parallel and both participate in identity — two blocks with an identical byte layout but different member *names* are **not** the same block, because the by-name binding resolves against those names.

Constant offsets are stored as 16-bit byte offsets, which caps a single block at 64 KiB. That is not an arbitrary limit: it is the constant-block size ceiling the shader model itself imposes, so storing a wider offset would buy nothing.

## `construct from reflection`

**Contract** — builds the block from a compiled program's description of it: reads the name, class, byte size and the full member list, computes the layout fingerprint, allocates the device buffer as a dynamic, host-writable buffer of exactly that size, and allocates the matching host image. Starts dirty, so the first draw uploads it even if nothing was written.

**Notes** — Debug builds tag the device buffer with the block's name so a graphics debugger shows something readable; incidental.

## `Similar`

**Contract** — true when two blocks are interchangeable: same name, same class, same layout fingerprint, same member count, byte-identical member type descriptions, and the same member names in the same order. The resource manager uses this to deduplicate — a new program's block that is `Similar` to an existing one is discarded in favour of the existing instance.

**Notes** — The fingerprint is checked before the byte comparison purely as a fast reject. The final name comparison is by interned-string identity, not by content, because all names come from the shared string pool.

## `Flush`

**Contract** — if dirty, map the device buffer with a *discard* hint, copy the whole host image over, unmap, and clear the dirty flag. Called once per draw for every block bound to any stage. Takes the submission context to map on, because a deferred context must not map a resource another context is mapping.

```text
FUNCTION flush(context)
  IF NOT dirty THEN RETURN
  target = map(device_buffer, context, discard_previous_contents)
  copy(target, shadow, size)
  unmap(device_buffer, context)
  dirty = false
```

**Notes** — Three decisions are folded into those five lines.

*Discard, not overwrite.* The driver is told the previous contents are worthless, so it can hand back fresh storage rather than stalling until the last draw that read the old contents has retired. A rebuild that maps without this hint will serialize the whole frame against its own constant uploads.

*Whole-buffer copy, no dirty range.* The source notes the absence of range tracking as a known gap. Since blocks are small and a mapped-with-discard buffer must be fully rewritten anyway (its previous contents are gone), partial upload would require keeping the shadow *and* re-copying the untouched parts — which is what this does. The decision is therefore correct as written, not merely unfinished; what is genuinely missing is the *upstream* check, that a write actually changed the value.

*One block instance per submission context.* Each context keeps its own copy of the block, created at the same time, because two contexts recording in parallel must not write the same device buffer. That is why destruction removes this block from every context's registry.

## destruction

**Contract** — unregisters from every submission context's constant-buffer registry before releasing the device buffer and the host image. Order matters: the registry holds raw references and would otherwise hand out a dead block.
