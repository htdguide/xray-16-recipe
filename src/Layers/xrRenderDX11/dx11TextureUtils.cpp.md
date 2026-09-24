# src/Layers/xrRenderDX11/dx11TextureUtils.cpp

> The pixel-format translation table between the engine's frozen format vocabulary and the device's, in both directions.

**Needs** — [`dx11TextureUtils.h`](dx11TextureUtils.h.md) · [Seam: Image codecs](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`dx11TextureUtils.h`](dx11TextureUtils.h.md)
**Tier floor** — T1: it names byte layouts of pixels.

## Purpose

The engine's format enumeration is the previous graphics generation's, because that is what the render-target declarations, the material files and the texture headers all speak. This file maps it to the device's and back. The reverse direction exists because the device answers questions about itself — the chosen back-buffer format, a target's actual format — in its own vocabulary, and the rest of the engine must hear the answer in its own.

## State

`Stateless.` A fixed table of pairs plus two linear lookups.

## `ConvertTextureFormat`

**Contract** — map one format either way; fail loudly on an unmapped value. The table is the interesting part, and four entries in it are decisions rather than transcriptions:

- **Channel order is not preserved.** The engine's four-channel colour formats are named in the opposite component order from the device's single mapped equivalent. The mapping is accepted as-is and the *shaders* compensate, which is why translated shader sources for a new API must apply the same swizzle.
- **A single-channel luminance texture maps to a single-channel red texture**, whose other components read as zero rather than replicating the value. The shipped shaders were written against replication, so every read of such a texture must swizzle. This affects light maps and mask textures throughout the game.
- **The 16-bit packed colour format has no device equivalent and is mapped to the full 32-bit one.** It survives only because one internal target declares it; the memory cost is accepted rather than adding a format-availability path.
- **Depth formats map to *typeless* layouts, not depth layouts.** A depth target must be readable as a texture later in the frame, and only a typeless allocation can carry both a depth view and a sampling view. The concrete depth interpretation is re-attached when the view is created; see [`dx11SH_RT.cpp`](dx11SH_RT.cpp.md).

**Notes** — The block-compressed formats are absent from the table entirely: compressed textures are never described through this vocabulary, because they arrive from the image file already identified and are handed to the device unchanged. That is the seam's "uploaded without decoding" requirement.

A synthetic format identifier is invented in the header for a 32-bit-depth-plus-8-bit-stencil layout that the old vocabulary had no name for. It is a four-character tag in the same space as the real ones, chosen so it cannot collide. If a rebuild defines its own format enumeration, this whole problem disappears — which is the honest advice, since the enumeration is engine-internal everywhere except inside texture file headers.
