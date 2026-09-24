# src/utils/xrLoadSurface.cpp

> Loads a texture off disk as a flat block of 32-bit pixels for a tool that needs to look at the image rather than upload it — and is currently built by nothing.

**Needs** — [`xrCore/LocatorAPI.h`](../xrCore/LocatorAPI.h.md) · [`xrCore/xrCore.h`](../xrCore/xrCore.h.md) · [Seam: Image codecs](../../SYSTEM-REQUIREMENTS.md#seam-image-codecs) · [Data: Textures](../../SYSTEM-REQUIREMENTS.md#5-data-and-persistence)

**Used by** — nothing in this recipe; entry point or dead code.

**Tier floor** — T1: it hands back a raw pixel block the caller owns and must release, and it depends on the decoder's row layout being contiguous.

## Purpose

The engine never decodes a texture: it reads the container's header and hands the
compressed blocks straight to the graphics device
([Seam: Image codecs](../../SYSTEM-REQUIREMENTS.md#seam-image-codecs)). Tools cannot do
that — a level compiler sampling a surface's colour, or an editor showing a thumbnail,
needs actual pixels. This file is that path: name in, decoded pixel block out.

**Nothing builds it.** The one build description that ever referenced it has the reference
commented out, so the file compiles nowhere and the three entry points it declares are
called from nowhere in this repository. It is kept because the *decision* it records is
still the right one for any tool that needs pixels, and because the last line of
`Surface_Detect` is a piece of policy nobody else states.

## State

```text
RECORD FormatRegistry
  extensions : list<text>      # lowercase, no leading dot, deduplicated case-insensitively
```

**Invariants**

- The registry is **advisory only**. It is built at start-up by asking the decoder which
  extensions it handles, and it is reported to the log — but nothing consults it when
  loading. It exists so an operator can see what the linked decoder supports.
- One extension is registered by hand ahead of the decoder's own list, because the game's
  source art uses it and the decoder announces it under a different spelling. Registering
  both is harmless; the registry deduplicates case-insensitively.

## `Surface_Init`

**Contract** — reports the decoder's version, seeds the registry with the hand-registered
extension, and then asks the decoder for the extension list of each container format the
tool cares about, splitting each comma-separated answer into individual entries. Reports
the total. Allocates; the registry owns every entry until the process ends. Call once.

```text
FUNCTION initialize_surface_loading()
  report(decoder_version())
  registry.add("tga")
  FOR EACH format IN the tool's list of container formats
    FOR EACH extension IN split(decoder.extensions_for(format), ",")
      registry.add(extension)          # ignores a case-insensitive duplicate
  report(registry.count)
```

**Notes**

- The format list is enumerated by name rather than iterated, so a decoder that gains a
  format does not gain an entry here. Since the registry is advisory, that costs nothing.

## `Surface_Detect`

**Contract** — resolves a texture's logical name to a path under the game's texture root
and reports whether a file exists there. **Only ever tries one extension** — the game's
own compressed-texture container. Returns the resolved path to the caller even on failure.

```text
FUNCTION detect_surface(name, OUT path) -> bool
  path <- resolve(root "$game_textures$", name + ".dds")
  RETURN file_exists(path)
```

**Invariants**

- **This is the policy worth keeping**: a texture is named without an extension everywhere
  in the game's data, and the *one* on-disk form is the compressed container. The format
  registry built above lists a dozen other formats and none of them is ever looked for.
  So a tool loading a texture loads the shipped one, not a source file that happens to sit
  beside it — which is what makes a tool's view of a surface match the engine's.
- Existence is tested by opening the file rather than by a metadata query, because the
  texture root may be served out of an archive where a metadata query means something
  different. A rebuild should ask the virtual filesystem, not the operating system —
  as written, this one asks the operating system and therefore cannot see a texture that
  lives inside an archive. That is a real limitation of the path and the reason it only
  ever worked against an unpacked data folder.

## `Surface_Load`

**Contract** — takes a texture name, strips any extension the caller left on it, resolves
it, decodes it, converts it to 32 bits per pixel if it is not already, and returns a
freshly allocated block of pixels together with the image's dimensions. Returns nothing
when the file does not exist or the decode produced nothing usable. **The caller owns the
block and must release it.** Mutates the name it is given — the extension is stripped in
place.

```text
FUNCTION load_surface(name, OUT width, OUT height) -> optional<bytes>
  strip_extension_in_place(name)
  IF NOT detect_surface(name, path) THEN RETURN none

  image <- decode(path)
  IF image.bits_per_pixel != 32 THEN image <- image.to_32_bits()
  IF NOT image.valid THEN RETURN none

  width  <- image.width
  height <- image.height
  block  <- allocate(width * height * 4)
  copy(block, image.first_row, width * height * 4)
  RETURN block
```

**Invariants**

- **The output is always four bytes per pixel**, whatever the source was, because the
  callers index it as a flat array of pixels. The conversion is unconditional in effect
  even though it is guarded by a test.
- **The copy assumes the decoded rows are contiguous and in the decoder's natural
  order** — it takes the address of the first row and copies the whole block in one go,
  never walking row by row. A decoder that pads rows to an alignment, or that returns
  bottom-up rows, produces a skewed or vertically flipped result with no error. This is the
  sharpest hazard in the file and a rebuild should copy row by row.
- The validity check happens **after** the conversion, so a file that failed to decode is
  converted before it is rejected. Harmless, and inverted from the order a rebuild wants.

**Notes**

- Stripping the extension in place mutates the caller's buffer, which means a caller
  cannot pass a literal and cannot reuse the name afterwards. It is a symptom of the
  original's allocation avoidance, not a decision.
- The whole file is one honest answer to "how does a tool get pixels": ask the decoder,
  normalize to one pixel format, hand back a block. Everything else on the page is the
  cost of doing that without owning a buffer type.
