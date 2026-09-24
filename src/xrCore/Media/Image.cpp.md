# src/xrCore/Media/Image.cpp

> Holds a raster in memory and writes it out as an uncompressed true-colour image.

**Needs** — [`Image.hpp`](Image.hpp.md) · [`FS.h`](../FS.h.md) · [`xrMemory.h`](../xrMemory.h.md)
**Used by** — [`Image.hpp`](Image.hpp.md)
**Tier floor** — T1: an 18-byte header whose field widths and packing are part of the file format, and a row loop that walks raw pixel memory.

## Purpose

Two jobs, joined because they share the pixel buffer: owning a raster (dimensions, format, and whether the buffer must be released with the object) and serializing one in the simplest container that every tool reads. The compressed side lives in [`ImageJPEG.cpp`](ImageJPEG.cpp.md) so that the whole lossy codec — and its optional third-party dependency — can be absent without taking this file with it.

## State

```text
RECORD Image
  format     : { unknown, three_channel, four_channel }
  channels   : int            # invariant: 3 for three_channel, 4 for four_channel;
                              #   set once at construction or at decode, never after
  width      : int
  height     : int
  pixels     : bytes          # row-major, top row first, stride == width * channels
  owns_pixels: bool           # true only when this object allocated the buffer
```

**Invariant** — `owns_pixels` is true exactly when the buffer came from decoding. A raster wrapped around a caller's buffer never frees it, which is what lets the screenshot path hand the renderer's readback staging memory straight in without a copy. This is the ownership question that a rebuild must answer explicitly, however its language spells it.

**Invariant** — rows are stored top-first and tightly packed. There is no stride field; anything that needs a different origin or padding converts on the way out.

## `SaveTGA`

**Contract** — writes the raster to an engine writer or to a host file path, in either the raster's own format or a requested one, with optional row padding to a four-byte boundary. Fails hard (an assertion, not a returned error) on an empty raster or an unsupported requested format: this is a developer-facing path, and a screenshot that silently produces nothing is worse than a stop. Writes are sequential and unbuffered beyond whatever the writer does.

The header is eighteen bytes, packed with no padding, little-endian throughout:

```text
RECORD Header               # written verbatim, 18 bytes, no alignment gaps
  id_length      : int (8-bit)    # 0 — no identification field follows
  colormap_type  : int (8-bit)    # 0 — no palette
  image_type     : int (8-bit)    # 2 — uncompressed true colour
  colormap_start : int (16-bit)   # 0
  colormap_len   : int (16-bit)   # 0
  colormap_depth : int (8-bit)    # 0
  x_origin       : int (16-bit)   # 0
  y_origin       : int (16-bit)   # 0
  width          : int (16-bit)
  height         : int (16-bit)
  bits_per_pixel : int (8-bit)    # 24 or 32
  descriptor     : int (8-bit)    # see below
```

The descriptor byte is where the two output formats differ and where the one deliberate oddity lives. Bit 5 (value 32) declares the rows to be stored **top-to-bottom**, which matches the in-memory layout and saves a reversal. For the four-channel form the low nibble is additionally set to 15, declaring fifteen attribute bits — which is wrong for an eight-bit alpha channel and should be 8. It is what the shipped writer emits and what the engine's own tooling expects to read back, so it is preserved; a rebuild that "fixes" it changes bytes some readers key on. For the three-channel form the low nibble is zero, which is correct.

```text
FUNCTION save_uncompressed(dst, requested_format, pad_rows)
  header <- as above, width and height from the raster
  IF requested_format is three_channel
    header.bits_per_pixel <- 24
    header.descriptor     <- 32                    # top-down, no attribute bits
    write header
    pad <- pad_rows ? (4 - (width * 3 mod 4)) : 0  # zero bytes appended per row
    FOR EACH row
      FOR EACH pixel
        write its first three channel bytes        # the fourth, if any, is dropped
      write pad zero bytes
  ELSE IF requested_format is four_channel
    header.bits_per_pixel <- 32
    header.descriptor     <- 32 OR 15              # top-down; the 15 is inherited
    write header
    IF the raster is already four_channel
      write the whole pixel buffer in one call     # the fast path: no per-pixel work
    ELSE
      FOR EACH pixel
        write its three channel bytes then a fully opaque alpha byte
  ELSE
    FAIL WITH UnsupportedFormat
```

**Notes** — the padding expression takes no remainder when the row is already aligned: `4 - (w*3 mod 4)` yields 4, not 0, for an aligned row, so an aligned row gets four surplus bytes. Whether that is intended is not recoverable from the source; padding is only requested by callers writing into fixed-pitch buffers, and those callers tolerate it. A rebuild should decide deliberately.

Both destinations — an engine writer and a host file — are served by one row loop parameterized on a write step. The duplication a rebuild might expect (one loop per destination) is the thing this arrangement exists to avoid; the choice of destination is not a decision, only the row layout is.
