# src/Layers/xrRender/ETextureParams.cpp

> The texture sidecar: how the shipped art says "this surface is metal", "pair me with this detail texture at this scale", "my bump map is this file and it is five centimetres deep".

**Needs** — [`ETextureParams.h`](ETextureParams.h.md) · [`xrCore/xr_token.h`](../../xrCore/xr_token.h.md) · [Seam: Image codecs](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs)
**Used by** — [`ETextureParams.h`](ETextureParams.h.md)
**Tier floor** — T1: the parameter chunk is read as a byte image from a shipped file.

## Purpose

Every texture in the game ships with a sidecar file describing it. The image itself carries pixels; the sidecar carries everything the renderer needs to know that pixels cannot express — which detail texture layers over it, which bump map accompanies it, how the surface reflects light, and how the image was built. **Frozen**: these files ship with all three games and the material system reads them at load.

This is the file the [texture description manager](TextureDescrManager.cpp.md) parses, and it is one of the two conventions §5 of the requirements calls out as "specific to this engine and must be reproduced exactly for the shipped art to look right".

## What the record says

```text
RECORD TextureParams
  # --- what the engine uploads ---
  format          : one of DXT1 / DXT1-with-alpha / DXT3 / DXT5 /
                           16-bit 4:4:4:4 / 1:5:5:5 / 5:6:5 /
                           32-bit RGBA / 8-bit alpha / 8-bit luminance /
                           16-bit alpha+luminance
  width, height   : int          # of the SOURCE image, not the shipped one
  flags           : bit set      # see below
  mip_filter      : int          # which of fifteen resampling kernels BUILT the mip chain

  # --- the detail pairing ---
  detail_name     : text         # the detail texture layered over this one
  detail_scale    : real         # how many times it tiles per world unit

  # --- the surface's reflectance ---
  material        : one of OrenNayar<->Blinn / Blinn<->Phong /
                           Phong<->Metal / Metal<->OrenNayar
  material_weight : real         # where between the pair this surface sits

  # --- the bump pairing ---
  bump_mode           : none / use / use with parallax
  bump_name           : text     # the height-and-normal texture
  bump_virtual_height : real     # metres; the parallax displacement depth
  ext_normal_map_name : text     # an override normal map, for a bump-map texture

  # --- build-time only, read by the tools, ignored by the engine ---
  type            : image / cube map / bump map / normal map / terrain
  border_color, fade_color, fade_amount, fade_delay : the mip-chain fade
```

## The material class — four *pairs*, not four models

**Invariants**

- The material field names a **pair of reflectance models**, and the weight says where between them this surface sits. Four pairs arranged in a cycle — rough-diffuse to soft-specular, soft-specular to sharp-specular, sharp-specular to metallic, metallic back to rough-diffuse — give a continuous two-dimensional space of surfaces addressed by one enumeration and one number.
- This is how the shipped art expresses "wet concrete" or "worn steel" with two numbers, and it is the input to the deferred renderer's material lookup: the pair index selects a row of a small lookup texture and the weight selects a column. A rebuild that substitutes a physically-based roughness/metalness pair must *map* these two values onto it, and the mapping is not the identity — the four pairs are not orthogonal axes.
- The default is the Blinn-Phong pair at weight zero, which is the plain diffuse-plus-specular surface most of the game is made of.

## The bump convention

**Invariants**

- A bump texture is **not** a normal map. It is a separate image whose channels carry the height and the derived normal in the engine's own packing; see [`Texture.cpp`](Texture.cpp.md) for what the renderer does with it. The `ext_normal_map_name` field exists to override the derived normal with an authored one, and is only meaningful when the record describes a bump texture rather than a colour one.
- `bump_virtual_height` is in **metres** and defaults to 0.05 — five centimetres. It is the depth the parallax displacement pretends the surface has. The default is a plausible depth for brick and plaster, which is most of what the game's parallax surfaces are.
- The bump mode is **clamped up to "none" on load** when the stored value is below it. The enumeration's zero value is a reserved slot from an abandoned automatic-normal-generation feature, and sidecars written before it was removed still carry it. This is a compatibility fix-up against shipped data and a rebuild must reproduce it.

## The detail pairing

**Invariants**

- A texture names its own detail texture and the scale at which it tiles. This is *per texture*, not per material: the same material template over two different ground textures gets two different detail layers automatically. That is the convention the shipped art depends on, and it is why the detail layer has no authoring surface of its own.
- Two independent flags say what the detail texture is *used as*: a diffuse multiply, or a bump contribution on the newer renderers. Both may be set. Neither being set means the pairing is recorded but unused.
- The scale defaults to 1 and is authored between 0.1 and 10000 — it is a tiling frequency, so large values are normal.

## The flags

```text
generate_mipmaps        # build-time: was a mip chain built
binary_alpha            # build-time: alpha is a cutout, not a gradient
alpha_border,
color_border            # build-time: clamp the edge to a border colour
fade_to_color,
fade_to_alpha           # build-time: fade the deepest mips toward a colour
dither_color,
dither_each_mip_level   # build-time: dithering during quantization

diffuse_detail          # RUNTIME: the detail texture multiplies the diffuse
implicit_lighted        # RUNTIME: this surface carries baked lighting
has_alpha               # RUNTIME: the SOURCE image had an alpha channel
bump_detail             # RUNTIME: the detail texture contributes bump
```

**Invariants**

- The flag bits are **not contiguous**: the build-time flags occupy the low bits and the four runtime flags sit at bits 23 through 26. The gap is where an obsolete greyscale flag and several removed options lived. The positions are frozen by the shipped sidecars and must be reproduced exactly; the gap has no meaning beyond history.
- **Two different alpha questions**, and confusing them is a real bug. `has_alpha` asks whether the *source art* had an alpha channel — authoring provenance. `has_alpha_channel()` asks whether the *shipped format* can represent one, which is a property of the format enumeration alone. The renderer needs the second to decide whether to enable alpha blending; the tools need the first to decide what formats are offered.

## `load(reader)`

**Contract** — reads a sidecar. The first chunk is required and every other is optional, so that a sidecar written by an older tool still loads. Never fails on a missing optional chunk; fails hard if the required one is absent.

```text
FUNCTION load(reader)
  REQUIRE the parameter chunk exists
  read format, flags, border colour, fade colour, fade amount,
       mip filter, width, height          # as a fixed byte sequence

  IF a texture-type chunk exists       read the type
  IF a detail chunk exists             read the detail name and scale
  IF a material chunk exists           read the material pair and weight
  IF a bump chunk exists               read height, mode (clamped), bump name
  IF an external-normal-map chunk      read that name
  IF a fade-delay chunk exists         read the delay
```

**Invariants**

- The required chunk's fields are read **in a fixed order with fixed widths**, not as a tagged list. The optional chunks are separately tagged. That mixture — a frozen fixed header plus versioned optional extensions — is the format's entire versioning strategy, and it is why fields added later (texture type, material, fade delay) each got their own chunk rather than extending the header.
- The format enumeration is read as a raw value with no validation. A sidecar naming a format this build does not know produces an out-of-range value that surfaces later as a failed upload.

## `save(writer)`

**Contract** — writes every chunk unconditionally, including empty ones. A sidecar written by this engine always has all seven chunks.

**Notes** — Writing empty optional chunks is harmless and means a round-trip normalizes an old sidecar to the current shape. Only the tools write these.

## The token tables

**Contract** — the authored name for each enumeration value: the mip filters, the texture types, the formats, the material pairs and the bump modes.

**Notes** — Two of these tables are *not* complete inverses of their enumerations. The format table omits the 4:4:4:4 format and two vendor-specific height formats that the enumeration still defines — they were dropped from the authoring interface without being removed from the format, and sidecars naming them still load. That asymmetry is unrecoverable as intent but is real, and a rebuild reading a sidecar must handle values the name table cannot spell.

The mip-filter table's first entry is "Advanced", which is not a kernel at all but a marker meaning "the following options apply"; the actual kernels start at the second entry. The numeric values are not in table order — box is zero, cubic is one, point is two — which is a strong sign the enumeration mirrors a third-party image library's own constants rather than being chosen here.

## The tools-only surface

**Contract** — three operations exist only in the authoring build: filling a property sheet from the record, comparing two records field by field against a list of named fields, and estimating a texture's memory cost.

**Invariants** — The property sheet is where the record's *semantics* are visible: changing the texture type rewrites several other fields at once (a normal map forces a 32-bit format, a Kaiser mip filter and mip generation on; a terrain texture forces block compression and baked lighting on and everything else off). Those rewrites are the authoring rules that produced the shipped sidecars, which makes them the best available documentation of what each type means.

The memory estimate divides an uncompressed size by a per-format constant and multiplies a mip chain by three halves — the standard geometric-series approximation. It also looks for a sibling sequence file and multiplies by the frame count, because an animated texture is a sequence of images sharing one sidecar.
