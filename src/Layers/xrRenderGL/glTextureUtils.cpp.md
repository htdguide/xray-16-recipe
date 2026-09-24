# src/Layers/xrRenderGL/glTextureUtils.cpp

> The render-target format table: which of the engine's Direct3D 9 format names this backend can honour, which it quietly substitutes, and which it refuses.

**Needs** — [`glTextureUtils.h`](glTextureUtils.h.md) · [`CommonTypes.h`](CommonTypes.h.md) · [Seam: Image codecs](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs)
**Used by** — [`glTextureUtils.h`](glTextureUtils.h.md)
**Tier floor** — T1: each entry names an exact byte layout for a driver allocation.

## Purpose

The engine names every render target's format in Direct3D 9 terms, chosen when the deferred renderer was designed: a 64-bit fat buffer here, a single-channel float there, a two-channel half-float for the luminance chain. This table says what each of those becomes on this device.

The file answers the seam's question "what happens when a shipped format is unavailable" for the *engine-created* half of the texture population. The answer has three shapes and a rebuild should know all three, because two of them are lossy:

1. **Exact.** Most entries are a faithful rename — eight-bit four-channel, sixteen-bit two- and four-channel, the half-float and float families, and packed depth-stencil.
2. **Widened.** Two entries are substitutions rather than translations. The 5-6-5 packed colour format is served with a full eight-bit four-channel target, wasting memory to avoid a format this backend does not want to allocate; it survives only because the one target that asks for it is a placeholder. A depth format requested *without* stencil is served with the packed depth-plus-stencil format, since this backend allocates a single combined depth-stencil attachment anyway.
3. **Channel-swizzled.** The single-channel luminance format becomes a plain single-channel red target. On the old API, sampling a luminance texture replicated the one channel across red, green and blue; here it does not, and the shader must replicate it. The behaviour difference is invisible in the table and shows up as a black image if the shader is not adjusted — which is why the shipped GLSL shader set is a *fork* of the HLSL set rather than a mechanical translation of it.

Anything not in the table faults in a checked build and yields "no format" otherwise. A rebuild should keep that failure loud: a missing render-target format is a design error, not a device limitation to be routed around.

## State

```text
# An ordered association list, searched linearly. It has fewer than twenty
# entries and is consulted only at target creation, so the search cost is
# irrelevant; a rebuild should use whatever map its language offers.
map<EngineFormat, DeviceInternalFormat>
```

## `convert_texture_format(engine_format)`

**Contract** — returns the device internal format for an engine format name, or faults in a checked build and yields "no format" in a shipping build. Pure; allocates nothing.

**Notes** — What is *absent* from this table is as informative as what is in it. The block-compressed families (the DXT trio and the two-channel pair) do not appear, because a compressed texture is never created by the engine — it arrives already described by the image codec and is uploaded as blocks without decoding, which is the whole point of the arrangement (see [Seam: Image codecs](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs) and [`glTexture.cpp`](glTexture.cpp.md)). Likewise, the vertex-component formats that share the Direct3D 9 format namespace are handled by [`glBufferUtils.cpp`](glBufferUtils.cpp.md)'s own tables, not here — the two namespaces overlap in the original API and a rebuild should keep them apart.
