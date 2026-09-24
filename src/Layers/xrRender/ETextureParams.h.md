# src/Layers/xrRender/ETextureParams.h

> Declares the texture sidecar record — the per-texture authoring description that ships beside the image and tells the engine what the texture *means*.

**Needs** — [`ETextureParams.cpp`](ETextureParams.cpp.md)
**Used by** — [`ETextureParams.cpp`](ETextureParams.cpp.md) · [`TextureDescrManager.cpp`](TextureDescrManager.cpp.md) · [`TextureDescrManager.h`](TextureDescrManager.h.md)
**Tier floor** — T1: the record is written to and read from a shipped binary file and its first chunk is read as a byte image.

## Purpose

Declares the surface implemented in [`ETextureParams.cpp`](ETextureParams.cpp.md), and fixes the record's vocabulary — the enumerations, the flag bits and the chunk identifiers. Those are part of the shipped data format and are the substance of the file; see the implementation twin for what each means.

## Exported units

- **`TextureParams`** — the record: format, flags, border and fade settings, mip filter, source dimensions, detail-texture pairing, material class and weight, bump-map settings and names.
- **`ETType`** — what kind of texture this is: image, cube map, bump map, normal map, terrain.
- **`ETFormat`** — the compressed or uncompressed format the engine will upload.
- **`ETBumpMode`** — none, use, or use with parallax displacement.
- **`ETMaterial`** — the four reflectance blends a surface can be.
- **Mip filter identifiers** — fifteen resampling kernels, used only when building the texture, never at load.
- **`clear()` / `has_alpha()` / `has_alpha_channel()`** — reset, and the two distinct alpha questions.
- **`load(reader)` / `save(writer)`** — the frozen read and write of the sidecar.
- **The chunk identifiers and the thumbnail dimensions** — the file's own shape.
- **The token tables** — the authored names for each enumeration, which are what the tools show and what a rebuild needs to read a sidecar written by hand.

**Notes** — The record holds interned strings and is also zeroed wholesale on reset, which in the original requires destroying and reconstructing those strings around the zeroing. That is pure C++ bookkeeping: what survives is "reset means every field returns to its stated default, and the defaults are not all zero."
