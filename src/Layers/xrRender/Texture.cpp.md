# src/Layers/xrRender/Texture.cpp

> Turns a texture *name* into a device texture: where the file is looked for, how many top mip levels are dropped, and — when a bump map is missing — how one is synthesized from the diffuse texture in the exact channel packing the shipped shaders expect.

**Needs** — [`SH_Texture.h`](SH_Texture.h.md) · [`ResourceManager.h`](ResourceManager.h.md) · [`xrRender_console.h`](xrRender_console.h.md) · [Seam: Image codecs](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs) · [Seam: Graphics device](../../../SYSTEM-REQUIREMENTS.md#seam-graphics-device)
**Used by** — [`ResourceManager.cpp`](ResourceManager.cpp.md) · [`ResourceManager_Resources.cpp`](ResourceManager_Resources.cpp.md) · [`SH_Texture.cpp`](SH_Texture.cpp.md)
**Tier floor** — T1: it decodes a container header, allocates device surfaces, and writes individual bytes into locked surface memory in a fixed channel order that shipped shaders read back.

## Purpose

Every material names its textures as bare strings. This file is the only place that resolves such a string to pixels on the device. Three decisions live here and nowhere else:

1. **Where a texture comes from** — the search order across the virtual filesystem's roots, and what happens when nothing is found.
2. **How much of it is loaded** — the quality setting drops whole mip levels at load time, and some textures are singled out to be dropped harder.
3. **What a bump map *is*** — the channel packing, and the synthesis path that fabricates a conformant one from a diffuse texture when the artist did not author a bump map. This is the packing named as frozen in the system requirements; the shipped shaders sample it by channel and there is no header anywhere describing it. It is written down here.

## State

`Stateless.` It produces device textures and hands them to the resource manager, which owns them.

The two records it implicitly defines are the bump-map *pair*, which is a naming convention, not a file format:

```text
CONVENTION bump map pair, for a material texture named "<base>"
  "<base>_bump"    : the normal/gloss texture   (block-compressed, 4 channels)
     R = gloss
     G = normal Z          # note the order: the normal is stored REVERSED,
     B = normal Y          # X in the alpha channel, Z in red-adjacent
     A = normal X          # alpha is compressed independently, so X keeps its precision

  "<base>_bumpX"   : the error/height companion  (block-compressed, 4 channels)
     R = 2 * (compression error in normal X) + 128
     G = 2 * (compression error in normal Y) + 128
     B = 2 * (compression error in normal Z) + 128   # channels here are NOT reversed
     A = height                                       # drives parallax / steep parallax

  "<base>_bump#"   : a variant suffix; a bump map whose companion data differs
                     (the hash is part of the name, not a separator)
```

**Invariants**

- The `_bump` texture's channels are reversed and the `_bumpX` texture's are not. The asymmetry is not a mistake: the reversal exists to put normal X into the alpha channel of a block-compressed format, which stores alpha in its own, higher-precision block; the companion texture is never sampled as a normal, so it has no reason to be swizzled.
- The error channels are stored biased by 128 and scaled by 2. They are a *signed* quantity in an unsigned texture, and the factor of 2 spends the available range on the error magnitudes that actually occur — block compression of a normal map produces small errors. A shader reconstructing the high-precision normal computes `compressed + (error - 128) / 2` per channel.
- Height is the luminance of the diffuse texture when synthesized, and an authored channel when the artist supplied one.
- When the two textures of a pair disagree in dimensions or mip count, nothing detects it. Both are sampled with the same coordinates at the same mip, so they must match.

## `texture_load`

**Contract** — given a texture name, returns a device texture and reports the number of bytes of system memory the loaded image cost. Blocks on file I/O and on device allocation. Never returns without a texture in the game build unless even the placeholder is missing; the caller's reference is the only owner. Called from the resource manager on a cache miss, so it runs once per distinct name per level load, not per frame.

```text
FUNCTION texture_load(name) -> (Texture, bytes)
  key = name with a trailing ".tga", ".dds", ".bmp" or ".ogm" removed
  # the shipped material files name textures with and without extensions,
  # and with extensions of formats that were converted away years ago

  IF key is absent from $game_textures$ AND key contains "_bump"
    GO TO synthesize a bump map                  # the interesting path, below

  path = first that exists of:  $level$/key.dds
                                $game_saves$/key.dds
                                $game_textures$/key.dds
  IF none exists
    report the missing name
    path = $game_textures$/<placeholder>.dds     # a loud magenta-style stand-in
    FAIL WITH fatal IF the placeholder is itself missing,
      unless running in the oldest game's compatibility mode

  info = read the container header
  IF the header is unreadable
    RETRY with the placeholder                   # once; retrying the placeholder is an error
  IF the container is a cube map
    RETURN device cube texture built from it, all mips
  ELSE
    RETURN load_2d(path, info)
```

**Invariants** — the retry onto the placeholder asserts that the path it is retrying with is *different* from the one that just failed, which is the whole guard against an infinite retry loop. A rebuild should express that as a bounded retry rather than as a path comparison.

**Notes**

- Three search roots, in that order, and the order is load-bearing: a level may override a shared texture by shipping its own copy, and a save game may carry a texture (the screenshot embedded in a save's thumbnail lives there). The shared archive is last.
- The name is lowercased before the device is asked for the 2D path but not before the cube path. That asymmetry has no discoverable reason; a rebuild should normalize once, up front.
- The placeholder texture's name is a frozen path in the shipped data (under the editor-assets folder). The oldest of the three games ships without it, which is why its absence is only fatal for the other two.

### Loading a 2D texture — dropping mips at load

```text
FUNCTION load_2d(path, info) -> Texture
  staging = decode the container into system memory, keeping every mip
  skip    = load_lod(path)                 # 0, 1 or 2 — how many top mips to discard
  device  = copy_reducing(staging, skip)
  RETURN device
```

The engine never asks the decoder to skip levels. It decodes everything into a staging image and then copies mip *n+skip* of the staging image onto mip *n* of the device image, top-aligned, stopping when the device image runs out of levels. Dropping a level this way costs a full decode but is exact: no resampling happens, the smaller mips are the ones the artist authored.

```text
FUNCTION copy_reducing(source, skip)
  width, height, levels = source dimensions and level count
  WHILE levels > 1 AND skip > 0
    width = width / 2 ; height = height / 2 ; levels = levels - 1 ; skip = skip - 1
  clamp width and height to at least 1
  destination = device texture of (width, height, levels), same format
  copy source level (i + dropped) into destination level i, for every destination level,
    with filtering disabled                     # these are authored mips, not resampled ones
```

**Notes** — The loop refuses to reduce below a single mip level, so a texture with no mip chain is never shrunk. That is the reason a one-level texture also reports its full file size as its memory cost rather than a reduced one.

### `get_texture_load_lod` — which textures are reduced, and by how much

```text
FUNCTION load_lod(path) -> int in {0, 1, 2}
  IF path contains any substring listed in the configured "reduce lod" list
    # these textures are known to be oversized for what they show
    IF quality is the highest        -> 0, but 1 when address space is scarce
                                        and the oldest renderer generation is not running
    ELSE IF quality is mid           -> 1
    ELSE                             -> 2
  ELSE
    IF quality is one of the top two -> 0
    ELSE IF quality is mid           -> 1
    ELSE                             -> 2
```

**Notes**

- The reduction list is a section of the shipped configuration containing *substrings*, not names: an entry matches anywhere in the path, so a whole folder is reduced by naming its prefix. The list is content-tuned — these are textures the artists made larger than they needed to be — and a rebuild reads the same section.
- The address-space check is a 32-bit concern: dropping one more mip everywhere is the difference between fitting and not. On the oldest renderer generation it is skipped because that generation's working set is small enough not to need it. At 64 bits the check always passes and the branch is dead; a rebuild may delete it and keep the list.
- The list is scanned linearly for every texture. It is short and this runs at load time.

### `calc_texture_size` — accounting for what was dropped

```text
FUNCTION reported_size(skip, mip_count, file_bytes) -> int
  IF mip_count == 1 RETURN file_bytes           # nothing to drop, nothing to discount
  size = file_bytes
  REPEAT skip TIMES
    size = size - size / 1.333
  RETURN floor(size)
```

A full mip chain is four thirds of its top level, so the top level is `3/4` of the whole and removing it leaves `size - size/1.333`. The constant is that ratio, written as a decimal. This is an *estimate* used for memory reporting and for the texture budget, not an allocation size — it is computed from the compressed file length, which is only proportional to the decoded size for block formats. Every shipped texture is a block format, so it is close enough.

### Synthesizing a bump map

Reached when a material asks for `<base>_bump` and no such file exists. The rule is: never fail, always produce something with the right channel layout, and say so in the log so an artist can fix it.

```text
FUNCTION synthesize_bump(key)
  IF key contains "_bump#"
    RETURN the shipped dummy variant-bump texture      # a flat, neutral pair
  IF key contains "_bump"
    RETURN the shipped dummy bump texture
  # (the game build always takes one of the two above; the derivation below is
  #  what the authoring tools do, and is the definition of the packing)

  base = load "<key with _bump removed>" as uncompressed 4-channel
  normal = compute a normal map from base's luminance,
           with bump height 8, also writing ambient occlusion into alpha
  repack normal in place:                       # gloss, then reversed normal
      R = (occlusion / 3 + 8 * 3) / 4           # a default gloss; see below
      G = old B  (normal Z)
      B = old G  (normal Y)
      A = old R  (normal X)
  normal_compressed = block-compress with an alpha block, reducing by load_lod(base)

  IF the renderer generation supports the companion texture
      restored = decompress normal_compressed back to 4 uncompressed channels
      error    = per channel, 128 + 2 * (normal - restored)
      repack error:                             # un-reverse, and take height from base
          R = error's A  (normal X error)
          G = error's B  (normal Y error)
          B = error's G  (normal Z error)
          A = mean of base's R, G and B         # height
      register the result as a device texture named "$user$/<key>_bumpX"
  RETURN normal_compressed
```

**Notes**

- **Bump height 8.** The normal map is computed by treating the diffuse texture's luminance as a height field scaled by this factor. It is a look decision — the amplitude of the fake relief — and every synthesized bump map in the game shares it. There is no per-texture override.
- **The default gloss is almost a constant.** Occlusion is divided by three and then averaged against a constant 8 with weight one to three, which lands every texel between 6 and about 27 out of 255 — a dull surface with a faint hint of the occlusion pattern. The intent is clearly "make it not shiny, but not perfectly flat either"; the exact weights are taste.
- **The companion texture is generated only for the newer renderer generations.** The oldest one does not sample it, so computing it would be wasted work — and the generation check happens at build time in the original, which a rebuild turns into a capability query.
- **The companion is registered under the writable user root, not written to disk.** It is a synthesized resource that must be reachable by name, because the material system binds it by name like any other texture; giving it a name under a root nothing else uses is how a synthesized resource enters the name space. The registered texture takes ownership immediately, which is why the local reference is dropped right after.
- The error texture is computed by compressing and then *decompressing* the normal map and subtracting. That is the only honest way to know the error: it depends on the block compressor's choices, which are not predictable from the input. A rebuild using a different compressor gets different error values, and that is correct — the pair is self-consistent, not portable. But a rebuild that ships the *original's* `_bumpX` files must use them as they are, since they encode the original compressor's error.

### The per-texel rewrite helpers

**Contract** — walk every mip of a destination texture in lockstep with one or two source textures of identical dimensions and format, applying a per-texel function. Assert that the level counts match and that the format is the uncompressed four-channel one, because the packing functions address channels by name.

**Notes** — Two variants exist, one-source and two-source, and both are the same loop. They are separate because the packing functions have different arities; the split is incidental. What survives is the requirement: channel repacking happens on *uncompressed* data, across *all* mips, before compression — repacking after compression would destroy the block structure.

## Could not recover

Two packing functions are defined here and called from nowhere: one that takes gloss from a *second* texture's alpha rather than defaulting it, and one that takes height from a second texture's red channel rather than from luminance. They are the "the artist authored a gloss/height source" half of the synthesis path, and the call site that used them is gone. They are worth keeping in a rebuild's notes because they document the *intended* authoring pipeline: gloss in the alpha of a dedicated map, height in its red channel.
