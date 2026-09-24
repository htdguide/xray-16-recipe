# src/Layers/xrRenderDX11/dx11Texture.cpp

> Loads a texture from the virtual filesystem: where it is looked for, what it falls back to when missing, how many top mip levels are dropped, and how a stubborn file is retried.

**Needs** — [`dx11TextureUtils.h`](dx11TextureUtils.h.md) · [`dx11HW.h`](dx11HW.h.md) · [`xrCore/FS.h`](../../xrCore/FS.h.md) · [Seam: Image codecs](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs)
**Used by** — [`dx11SH_Texture.cpp`](dx11SH_Texture.cpp.md) · [`r4_rendertarget_build_textures.cpp`](../xrRenderPC_R4/r4_rendertarget_build_textures.cpp.md)
**Tier floor** — T1: it hands a block-compressed mip chain to the device without decoding it.

## Purpose

One function with four decisions in it, each of which a rebuild will otherwise get wrong: **where a texture file is found, what happens when it is not, how much of it is skipped under a memory budget, and what to do when the driver refuses it.**

## `texture_load`

**Contract** — takes a logical texture name, returns a device texture and reports the host-memory size attributable to it. Blocks on file input. Returns nothing rather than failing when the texture and all its fallbacks are missing. Called during level load from more than one thread.

```text
FUNCTION load_texture(name) -> (texture, accounted_size)
  strip any image extension the caller supplied      # names in data carry stale extensions
  # --- location ---
  IF the name looks like a bump map AND is not found THEN
    substitute the default flat bump map, choosing between the two variants
    by whether the name carries the second-channel marker
  ELSE
    search, in order: the current level's own directory,
                      the save directory,
                      the shared texture directory
    IF still missing THEN substitute the "missing texture" placeholder
                          and report the name

  bytes = read whole file
  skip = mip_levels_to_skip(name)

  FOR attempt IN 1..3
    image = parse container(bytes, permissive)
    IF skip > 0 AND NOT a cube map THEN
      halve width and height, drop leading mip levels, `skip` times,
      never below one texel
    IF the format is block compressed THEN
      round width and height up to a multiple of the block edge
    texture = device.create_immutable(image starting at the skipped level)
    IF created THEN
      accounted_size = size_after_skipping(skip, mip_count, file_size)
      RETURN
    relax a container-interpretation option and retry     # see Notes
  report failure
```

**Invariants** — Cube maps are never mip-skipped: their faces must stay square and consistent, and the engine's cube maps are small.

Block-compressed dimensions are rounded up to the block edge. The shipped data contains textures whose declared size is not a multiple of the block size, and a driver that validates this strictly will refuse them.

**Notes** — *The three-attempt retry* is a compatibility ladder, not a loop: each attempt relaxes one assumption about how the container's older-style headers should be interpreted — first assuming full support for 16-bit packed formats, then without them, then forcing everything to an expanded layout. Older drivers reject the earlier interpretations. The ladder is a fact about the shipped data meeting real drivers, and a rebuild that supports only the strict interpretation will fail to load some of the game's textures.

*The fallback chain is ordered and load-bearing*: a level may override a shared texture by shipping its own copy of the same name, and the save directory sits between them so that a saved game's captured screenshot can be loaded as a texture by name.

*The bump-map fallback* substitutes a flat default rather than the generic missing-texture placeholder, because a missing bump map with a pink placeholder would make the surface look violently wrong, while a flat default simply means "no bumps". The two variants correspond to the engine's two-part bump encoding.

## `get_texture_load_lod`

**Contract** — decides how many of a texture's largest mip levels to skip, from the player's texture-detail setting and a configured list of texture names that are demoted one step earlier than everything else.

```text
FUNCTION mip_levels_to_skip(name) -> 0 | 1 | 2
  IF the name matches an entry in the demotion list THEN
    detail setting lowest   -> 0 normally, 1 when address space is tight
    detail setting middle   -> 1
    otherwise               -> 2
  ELSE
    detail setting below the middle -> 0
    detail setting below the top    -> 1
    otherwise                       -> 2
```

**Notes** — The demotion list is configuration data, not code: it names texture-path fragments whose textures are large and rarely seen close up. The address-space check matters only on a 32-bit build, where the whole texture set will not fit in the process's virtual address space at full detail; on 64 bits it is always satisfied.

## `calc_texture_size`

**Contract** — estimates the memory a texture accounts for after skipping levels. A single-level texture keeps its file size; otherwise each skipped level removes three quarters of what remains, which is the limit of the geometric series a full mip chain forms.

**Notes** — The divisor is expressed as a constant of about 1.333, which is the same 4/3 relationship written the other way round. The figure is an accounting estimate for the memory display, not an allocation.

## `fix_texture_name`

**Contract** — strips a trailing image or video extension from a texture name. The game's data refers to textures with and without extensions, and with several different ones, so the extension is discarded and the loader appends the one it actually reads.
