# src/Layers/xrRenderGL/CommonTypes.h

> The translation dictionary: every Direct3D 9 type name the shared renderer core still speaks, redefined as this backend's own handle, record or enumerant.

**Needs** — [`glState.h`](glState.h.md) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`glBufferUtils.cpp`](glBufferUtils.cpp.md) · [`glHW.cpp`](glHW.cpp.md) · [`glStateUtils.cpp`](glStateUtils.cpp.md) · [`glTextureUtils.cpp`](glTextureUtils.cpp.md) · [`stdafx.h`](../xrRenderPC_GL/stdafx.h.md)
**Tier floor** — T1: it fixes the width and meaning of opaque device handles and the byte shape of a viewport record handed to a driver.

## Purpose

The renderer core in [`xrRender`](../xrRender/README.md) was written against a Direct3D 9 device and never stopped naming its types that way — buffers, viewports, comparison functions, queries and state blocks all carry those names throughout tens of thousands of lines that both backends share. Rather than rewrite those call sites, each backend supplies a header that gives the D3D9 vocabulary a local meaning. This is that header for the OpenGL backend, and it is the cheapest half of the translation: everything here is a rename decided at build time, costing nothing at run time.

The expensive half lives next door. Where a D3D9 enumerant's *values* cannot be chosen freely — blend factors, stencil operations, address modes, texture formats — a run-time lookup is needed, and those live in [`glStateUtils.cpp`](glStateUtils.cpp.md) and [`glTextureUtils.cpp`](glTextureUtils.cpp.md). The dividing line is worth naming, because a rebuild gets to redraw it: any enumerant this header can define *as* the target API's own value is free; any enumerant whose D3D9 numbering is baked into shipped data or into the shared core's switch statements needs a table.

A rebuild that defines its renderer interface afresh deletes this file entirely. It exists only because the interface was shaped by a different API.

## State

`Stateless.` — the file declares types, not data.

```text
# Device object handles. The engine only ever stores, compares against zero,
# and hands these back to the device; it never dereferences them, which is
# exactly why an integer name works where the other backend uses a pointer.
VertexBufferHandle   : int (opaque device name, 0 == none)
IndexBufferHandle    : int (opaque device name, 0 == none)
ConstantBufferHandle : int (opaque device name, 0 == none)
HostBufferHandle     : reference to host memory        # the staging side

RECORD Viewport
  top_left_x, top_left_y : int
  width, height          : int
  min_depth, max_depth    : real   # clamped to [0,1]

ENUM ClearFlag   { depth, stencil }          # colour is always implied separately
ENUM QueryKind   { event, occlusion }
ENUM ComparisonFunc
  never, less, equal, less_equal, greater, not_equal, greater_equal, always
```

## `ComparisonFunc`

**Contract** — the depth, stencil and sampler-comparison predicates. Defined so its members *are* the target API's own comparison enumerants, so a depth or stencil function set by the material system passes through to the device with no lookup. Everything that reaches this enum from shipped material data has already been mapped to it by [`glStateUtils`](glStateUtils.cpp.md)'s converter; the two exist together because the *engine-facing* name and the *device-facing* value coincide here and do not coincide for blend factors.

## `Viewport`

**Contract** — a target-relative rectangle plus a depth range. The one trap a rebuild must know: this API's window origin is bottom-left while the interface's heritage assumes top-left, so every rectangle that crosses this boundary — viewport, scissor, clear rectangle — has its vertical coordinate mirrored against the current target height. The mirroring is *not* done here; it is done at each use site ([`glR_Backend_Runtime.h`](glR_Backend_Runtime.h.md)), which is a defect worth fixing in a rebuild by mirroring once, at the boundary.

## `VertexElement`

**Contract** — one entry of a vertex-layout description: stream index, byte offset, component type, semantic and semantic index. Aliased to the frozen Direct3D 9 declaration record because the *model and level formats ship vertex layouts in exactly that encoding* (see §5 of [SYSTEM-REQUIREMENTS](../../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)). This is the one type in the file that a rebuild cannot rename away: its field meanings are frozen by the game data, and it must be parsed as such no matter what the device's own layout description looks like. Converting it to device attribute bindings is [`glBufferUtils`](glBufferUtils.cpp.md)'s job.

## `StateBlock`

**Contract** — the type the shared material compiler stores an interned render-state block as. Bound to [`glState`](glState.cpp.md), this backend's implementation.

**Notes** — The header also supplies a do-nothing form for statements that exist only on the Direct3D 11 path, so shared code can name them unconditionally. That is a C++ convenience for keeping one source tree; a rebuild expresses it as a capability query or as two implementations of an abstract port, and nothing is lost.
