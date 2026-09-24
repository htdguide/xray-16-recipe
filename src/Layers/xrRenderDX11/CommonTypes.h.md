# src/Layers/xrRenderDX11/CommonTypes.h

> The backend's version-neutral vocabulary: one name per device concept, mapped onto whatever the underlying graphics API calls it.

**Needs** — [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx11BufferUtils.cpp`](dx11BufferUtils.cpp.md) · [`dx11R_Backend_Runtime.h`](dx11R_Backend_Runtime.h.md) · [`stdafx.h`](../xrRenderPC_R4/stdafx.h.md)
**Tier floor** — T1: every name in it is a device handle type, a byte-layout description or a driver enumeration value.

## Purpose

Every other file in this backend is written against the names declared here rather than against the graphics API's own names. Historically this let one body of source compile against two generations of the same API; today it does something more useful, and it is the reason this file survives as a decision rather than as noise: **the backend has exactly one place where "what the device calls this" is written down**, and the rest of the module speaks the engine's own dialect.

A rebuild targeting a different graphics API reproduces this file as its *translation table* and leaves the other forty files structurally untouched. That is the whole point of the file, and it is worth honouring: the one-to-one aliasing is what makes the port a single-file problem instead of a forty-file one.

## State

`Stateless.` It declares names only.

## The vocabulary

The names divide into five groups, and a rebuild needs each group whether or not it spells them this way:

- **Resource handles** — buffer (one handle type serves vertex, index and constant use), texture in 1D/2D/3D form, the untyped resource a texture and a buffer both are, the three view kinds (readable-by-shader, render target, depth-stencil), the unordered-access view, the input layout, the four shader stages' program objects, the query object, the device and the submission context.
- **Descriptors** — the creation records for textures, buffers, queries, samplers, rasterizer/blend/depth-stencil state, views, and the mapped-memory record returned by a map.
- **Enumerations** — usage class (immutable / default / dynamic / staging), bind flags, host-access flags, map modes (including the two that matter for streaming: *discard* and *no-overwrite*), cull mode, fill mode, comparison function, stencil operation, blend factor and operation, texture addressing mode, filter, clear flags, format-support questions, query kinds, and the view dimensionalities.
- **Limits** — the sampler-slot count per shader stage and the maximum 2D texture edge, both of which the engine reads rather than hard-codes.
- **Stream handles** — the aliases for vertex, index and constant buffer handles and for a host-side pointer, which is the layer the geometry streaming code in chapter 18 is written against.

**Notes** — One alias is not cosmetic: a **vertex element** is still the *legacy fixed declaration record* from the previous API generation, while an **input element description** is the current one. The engine's data — model formats and the material system — describes vertex layouts in the legacy form, so that form is the on-disk truth and this backend converts it to the current form when a layout is created. A rebuild must keep the legacy description as its interchange format (it is frozen in the model files) and treat the API-shaped one as derived; see [`dx11BufferUtils.cpp`](dx11BufferUtils.cpp.md).

The viewport record is wrapped only so that integer pixel coordinates can be passed where the device wants floating point without a conversion warning at every call site — incidental.

The single-context marker macro declared here (`DX11_ONLY`) marks code that exists only on this backend, so that source shared with the older generation can be compiled both ways. In a rebuild there is no shared source and the marker vanishes.
