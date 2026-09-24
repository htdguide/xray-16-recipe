# src/Layers/xrRenderGL/glTexture.cpp

> Loading a shipped image: where the file is looked for, what stands in when it is missing, and how a container's blocks reach the device without being decoded.

**Needs** — [`glSH_Texture.cpp`](glSH_Texture.cpp.md) · [Seam: Image codecs](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`glSH_Texture.cpp`](glSH_Texture.cpp.md)
**Tier floor** — T1: block-compressed payloads are handed to the driver as raw bytes with no host-side decode.

## Purpose

Three things happen here and only the middle one is obvious.

The **search order** decides which copy of a texture a level sees: a level-local file shadows the shared library, which is how the shipped levels override a handful of surfaces. It is frozen by the data.

The **upload** is the straightforward part: parse the container, allocate immutable storage for the whole mip chain, then push each level — compressed levels as blocks, uncompressed levels as pixels. The seam's promise that block formats are "uploaded without decoding" is kept exactly here.

The **fallbacks** are the part a rebuilder will not anticipate. A missing texture does not fail; it resolves to a placeholder, and *which* placeholder depends on whether the name looks like a bump map. That matters because a bump map placeholder must be a valid flat normal, not a magenta checkerboard — substituting the wrong one makes every surface in the level shade as if it were crumpled.

## `load_image(name)`

**Contract** — finds, parses and uploads an image, returning the device texture handle, the memory it should be charged, and the texture kind. Returns nothing on a hard failure. Blocks on the file read. Allocates.

```text
FUNCTION load_image(name) -> (handle, charged_bytes, kind)
  FAIL IF name is empty
  base := name with any of the known image extensions stripped

  # --- locate
  IF base has no file in the shared texture library AND base names a bump map THEN
      # A bump map placeholder must be a FLAT normal, and there are two of
      # them because the engine distinguishes a bump map from its companion.
      path := the dummy bump map, in the companion form if the name asks for it
      FAIL IF that does not exist either
  ELSE
      FOR EACH root IN [ the level's own directory,
                         the save directory,
                         the shared texture library ]
          IF a file exists under root THEN path := it; BREAK
      IF none matched THEN
          log "can't find texture"
          path := the not-found placeholder; RETURN nothing if even that is absent

  # --- parse and upload
  file    := open(path)
  texture := image_codec.parse(file bytes)            # container, not pixels
  FAIL IF the container yielded nothing

  kind          := device target for the container's target kind
  device_format := the codec's translation of the container's format,
                   including its channel swizzle

  allocate one texture; bind it as `kind`
  set the level range to [0, level count - 1]

  # The swizzle is applied EXCEPT for single-channel red content, because the
  # font textures rely on the untouched red/alpha arrangement.
  IF device_format's external form is not single-channel red THEN
      set the texture's channel swizzle from device_format

  IF the container is 2D or cube THEN
      allocate immutable 2D storage for every level at the base extent
  ELSE IF the container is 3D or a cube array THEN
      allocate immutable 3D storage for every level at the base extent
  ELSE FAIL

  FOR EACH layer, face, level IN the container
      sub_target := the cube face when the container is a cube, else `kind`
      IF the container's format is block-compressed THEN
          upload the level's bytes as compressed blocks, giving the byte count
      ELSE
          upload the level's bytes as pixels, giving external format and type
      log any device error, naming the file, and continue

  close(file)
  charged_bytes := charge_for(load_detail_level(path), level count, file length)
  RETURN (handle, charged_bytes, kind)
```

**Invariants** — storage is allocated for the *whole* mip chain before any level is uploaded, and it is immutable. This is the modern allocate-then-fill discipline and it is what lets the compressed path work at all: the driver knows the exact format and extent of every level in advance, so a block payload can be memory-copied into place.

**Invariants** — the channel swizzle is part of the format, not of the data, and the *one* exception is single-channel red content. That exception exists for the font atlases, whose greyscale-plus-alpha arrangement the swizzle would destroy. This is the same swizzle question that appears in the format table ([`glTextureUtils.cpp`](glTextureUtils.cpp.md)) for engine-created single-channel targets, and the two are the same problem seen from opposite ends: the old API replicated a single channel across RGB and this one does not.

**Notes** — a per-level upload error is logged and the loop continues rather than aborting. A texture with one bad level renders with that level wrong; a texture that aborts the level load costs the player the level. The choice is right for a program that reads data it did not write.

## `load_detail_level(path)`

**Contract** — answers how many mip levels the user's texture-detail setting says to *charge* for this texture — 0, 1 or 2. Reads a configuration section listing textures whose detail should be reduced more aggressively than the rest, and matches the path against it by substring.

```text
FUNCTION load_detail_level(path) -> int
  IF path matches any entry in the reduce-detail list THEN
      RETURN 0 when detail setting < 1, 1 when < 3, else 2
  RETURN 0 when detail setting < 2, 1 when < 4, else 2
```

**Notes** — the two thresholds differ by exactly one step, so a listed texture drops a level earlier than an unlisted one across the whole detail range. Whether the specific pairs (1, 3) and (2, 4) were tuned or merely chosen is **not recoverable**; what is clear is the intent, which a rebuild can reproduce with its own numbers.

The more important fact is that **this only affects accounting, not the upload.** Every level is uploaded regardless. On this backend the detail setting does not reduce texture memory at all; it only reduces the number the memory display reports. That is a defect — the other backend skips levels — and a rebuild should skip levels for real.

## `charge_for(detail_level, level_count, file_length)`

**Contract** — estimates the video memory a texture occupies. A single-level texture is charged its whole file length. A mip chain is charged the file length reduced once per detail level by the ratio of a full mip pyramid to its tail — each step removes the fraction a full chain's top level represents.

**Notes** — the reduction constant is the mip-pyramid series ratio for two-dimensional textures, so one step removes the base level's share. It is an estimate over the compressed file size, not a measurement, and it is wrong for cube maps and arrays. Cube maps are exempted at the call site — their detail level is forced to zero — which suggests the author knew.

## `strip_image_extension(name)`

**Contract** — removes a trailing image extension from a name in place, for each of the four the game's data uses. Textures are named without extensions throughout the engine; the shipped material and model data sometimes includes one.
