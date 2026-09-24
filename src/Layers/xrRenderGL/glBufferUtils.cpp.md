# src/Layers/xrRenderGL/glBufferUtils.cpp

> Creates vertex and index buffers, and translates the frozen vertex-layout records that ship inside the game's models into this API's attribute bindings.

**Needs** — [`CommonTypes.h`](CommonTypes.h.md) · [`glR_Backend_Runtime.h`](glR_Backend_Runtime.h.md) · [`xrRender/BufferUtils.h`](../xrRender/BufferUtils.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`glR_Backend_Runtime.h`](glR_Backend_Runtime.h.md) · [`glResourceManager_Resources.cpp`](glResourceManager_Resources.cpp.md)
**Tier floor** — T1: byte strides and offsets are described to a driver, and dynamic buffers are mapped into the caller's address space.

## Purpose

Two jobs that happen to share a file.

The first is the buffer lifecycle the interface demands: immutable buffers built once from a staged copy, and dynamic buffers mapped and unmapped every frame with a discard hint. The shape of the staging buffer — *fill host memory, then upload once, then throw the host copy away* — is the load-bearing part, because it is what lets the level loader build geometry on a worker thread with no device context.

The second is the translation of a shipped vertex layout into device attribute state, and this is the more interesting one. Model and level files carry their vertex layouts as Direct3D 9 declaration records, frozen by the data. Three parallel tables turn each record's component type into a device component count, component type and normalization flag; a fourth turns its *semantic* into an attribute location. That fourth table is a contract with the shipped shader source, and it is the single most fragile joint in this backend.

## State

```text
RECORD StagingBuffer            # vertex or index; the two differ only in
  host_bytes    : optional<bytes>   #   which binding target they use
  device_handle : int               # 0 until the upload happens
  size          : int
  allow_readback: bool
# Invariants:
#   - the upload happens exactly once; a second flush is an error
#   - unless read-back was requested, the host copy is released at upload,
#     so the buffer's system-memory cost drops to zero once it is resident
#   - mapping a staging buffer does not touch the device at all: it hands back
#     a pointer into the host copy. That is what makes the level loader's
#     geometry build device-free and therefore thread-free.

RECORD StreamBuffer             # the per-frame ring the dynamic geometry uses
  device_handle : int
# Invariant: a stream buffer has no host copy. Mapping it maps device memory.
```

## Attribute location table

**Contract** — the semantic-to-location map every vertex layout is resolved through.

```text
position         -> 3        blend weight   -> unsupported
blend indices    -> unsupported
normal           -> 5        point size     -> unsupported
texture coord    -> 8        tangent        -> 4
binormal         -> 6        tessellation   -> unsupported
transformed pos  -> 3        colour         -> 0
fog              -> 7        depth, sample  -> unsupported
```

The element's semantic index is added to the base, so the second texture coordinate is location 9, the second colour is location 1, and so on.

**Invariants** — **these numbers are duplicated, by hand, in the shipped shader sources.** The GLSL shader set carries a header that defines the semantic names as exactly these integers so that a vertex program can declare its inputs at the matching locations. Change one side and the geometry silently arrives in the wrong attribute. A rebuild has two honest options: keep a single table and generate the shader-side definitions from it, or bind attributes by name at link time and delete the table. The original does neither, and a reader who does not know this exists will spend a long day on it.

Semantics marked unsupported are skipped rather than faulted: a shipped layout may carry blend weights that this renderer's skinning path does not read from the vertex stream.

## `convert_vertex_declaration(layout, declaration)`

**Contract** — builds the device-side attribute state for a layout, into a declaration object that already owns a vertex-array object. Binds the declaration, then for each element enables its attribute location and, *if the device supports separating attribute format from buffer binding*, records the element's format and which stream it draws from. Called once per distinct layout, at resource-creation time.

```text
FUNCTION convert_vertex_declaration(layout, declaration) -> ()
  backend.set_format(declaration)          # binds the vertex-array object
  FOR EACH element IN layout UNTIL a terminator stream index
    location := attribute_location(element.semantic) + element.semantic_index
    IF location IS unsupported THEN CONTINUE
    enable attribute at location
    IF device supports separate attribute format THEN
        describe format at location: component count, type, normalization, offset
        bind location to element.stream
```

**Notes** — Where the device does *not* support separating format from binding, the format is not recorded here at all; instead the whole layout is re-described every time a vertex buffer is bound (see `set_vertex_pointers` below). That is the fallback path, it costs one call per attribute per buffer bind, and it is why the capability is worth probing.

## `set_vertex_pointers(declaration)`

**Contract** — the fallback path: re-describes every attribute against the currently bound vertex buffer, using the layout's own computed stride. Called from the backend's vertex-buffer bind when the device lacks separate attribute formats.

**Invariants** — the stride comes from the layout's stream 0. Layouts that draw from more than one stream would need one stride each; the original computes only stream 0's, which is correct for every shipped layout and would be wrong for a multi-stream one.

## `create_buffer(data, size, dynamic, is_index)` (private)

**Contract** — allocates a device buffer of the given size, optionally seeded from host bytes, hinted as either write-once-draw-many or write-often. Index and vertex buffers differ only in which binding target they are created under — on this API that choice also determines which target the buffer must later be bound to, which is why the flag is carried through rather than inferred.

## `StagingBuffer.create(size, allow_readback)` · `.map(offset, size, read)` · `.unmap(flush)` · `.destroy`

**Contract** — `create` allocates *host* memory only, with no device involvement. `map` returns a pointer into that host memory at the requested offset; reading is refused unless read-back was requested at creation. `unmap` with no flush does nothing — the caller is still filling. `unmap` with a flush uploads the whole buffer to a freshly created immutable device buffer and, unless read-back was requested, releases the host copy. `destroy` releases both.

**Invariants** — uploading twice is an error. The buffer's identity for drawing purposes is the device handle, which is zero until the flush; `is_valid` means exactly "has been uploaded".

**Notes** — The memory accounting matters and is explicitly exposed: system-memory usage is the host copy if it still exists and zero otherwise, and video-memory usage is read back from the driver rather than assumed. A rebuild should keep that split, because the engine's memory display distinguishes them and a level's geometry is large enough for the difference to be visible.

## `StreamBuffer.create(size)` · `.map(offset, size, flush)` · `.unmap` · `.destroy`

**Contract** — the per-frame ring. `create` allocates a dynamic device buffer with no initial contents. `map` binds it and maps the requested range for writing, unsynchronized, and *additionally with a whole-buffer discard when the caller signals a flush*. `unmap` releases the range.

**Invariants** — this is where the interface's "map/unmap of dynamic buffers with a discard hint" is satisfied, and the two flag sets are the whole of it:

```text
flush  = write | unsynchronized | invalidate whole buffer   # wrapped to the start
append = write | unsynchronized                             # writing further along
```

The unsynchronized flag in *both* sets means the driver will not wait for in-flight draws that read the buffer. That is safe only because the ring's caller never hands back a region the GPU might still be reading: it walks forward through the buffer and signals the flush exactly when it wraps. A rebuild that keeps the unsynchronized behaviour inherits that obligation; a rebuild that drops it gets correctness for free and a stall per frame.

The appending case is marked in the original as wanting a sub-range update instead of a map; that is an optimization, not a semantic difference.

## `vertex_size_of(layout, stream)` · `declaration_length(layout)` · `vertex_size_of_packed_format(format)`

**Contract** — thin delegations to the shared vertex-format helper, which owns the frozen knowledge of how wide each declaration component is and how the old packed-format bitfield expands into a declaration. They are re-exported here because the buffer code is the only caller and the shared helper is API-agnostic.
