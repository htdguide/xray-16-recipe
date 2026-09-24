# src/xrCore/Media/ImageJPEG.cpp

> Bridges the raster to the lossy codec: routes the codec's output through an engine writer, turns its fatal errors into a recoverable failure, and makes the whole feature optional.

**Needs** — [`Image.hpp`](Image.hpp.md) · [`FS.h`](../FS.h.md) · [Seam: Image codecs](../../../SYSTEM-REQUIREMENTS.md#seam-image-codecs)
**Used by** — [`Image.hpp`](Image.hpp.md)
**Tier floor** — T1: it hands a foreign library raw scanline pointers into its own buffer and has to survive that library's abort-on-error convention.

## Purpose

Decode a lossy image into a raster, and encode a raster into one. Everything interesting here is about the *boundary*, not the compression: the codec at this seam wants to write to a sink it pulls from, wants to report failure by longjumping out of the call, and may not be present in the build at all. This file answers all three, and is separate from [`Image.cpp`](Image.cpp.md) precisely so that the third answer — absence — costs nothing.

## State

Stateless between calls. Two short-lived adapters exist for the duration of one encode or decode:

```text
RECORD ErrorTrap            # replaces the codec's default "print and exit"
  resume_point : continuation

RECORD Sink                 # feeds the codec's block-oriented output to a stream writer
  buffer : bytes[4096]      # the codec fills this; we drain it
  writer : Writer
```

**Invariant** — the sink's buffer is drained completely on every full-buffer callback and the *partial* remainder is drained exactly once at the end. Missing the final partial flush truncates the file by up to one buffer, which is the classic bug at this boundary.

## `OpenJPEG`

**Contract** — decodes from a reader's remaining bytes, or from an explicit byte range, into this raster. On success the raster owns a freshly allocated pixel buffer and its width, height, channel count and format describe it. On failure it returns false, having released whatever the codec allocated and left the raster untouched. If the build has no codec, it logs once and returns false — callers must treat a missing codec and a corrupt file identically.

```text
FUNCTION decode(src: bytes) -> bool
  install ErrorTrap
  ON codec error: tear the decoder down and RETURN false

  read the header
  width    <- header width
  height   <- header height
  channels <- header component count
  format   <- three_channel IF channels == 3, four_channel IF channels == 4
  request three-channel output                 # regardless of what the file holds
  begin decode
  pixels <- allocate width * height * channels
  WHILE scanlines remain
    hand the codec a pointer to row (channels * width * current_row) and let it fill it
  finish and tear down
  RETURN true
```

**Notes** — the output colour space is forced to three channels while the buffer is sized by the file's declared component count. For a greyscale source that over-allocates harmlessly; for a four-component source the stride the codec writes and the stride assumed by the row pointer disagree and the image comes out sheared. Every image this path is actually asked to decode is a three-channel loading screen, so the discrepancy has never bitten — but a rebuild should size the buffer from the *requested* output format, not from the file's, and that is a correction rather than a reproduction.

## `SaveJPEG`

**Contract** — encodes the raster to an engine writer at a quality clamped to 0..100, optionally emitting rows bottom-up (which is what a graphics device's framebuffer readback needs, since it hands back an origin-at-bottom image). Returns whether it succeeded. On codec failure it tears the encoder down and returns false rather than aborting. If the build has no codec, logs and returns false.

A build whose codec lacks the extended colour spaces cannot encode a four-channel raster at all and refuses up front; one that has them maps four channels onto a red-green-blue-alpha input space. There is no conversion path — the raster is handed over as it sits.

```text
FUNCTION encode(dst: Writer, quality: int, bottom_up: bool) -> bool
  clamp quality to 0..100
  install ErrorTrap
  ON codec error: tear the encoder down and RETURN false

  attach Sink{ writer: dst }
  declare width, height, channel count and the matching input colour space
  apply defaults, then the quality
  begin
  WHILE scanlines remain
    row <- bottom_up ? (height - 1 - current) : current
    hand the codec a pointer to pixels at row * width * channels
  finish and tear down
  RETURN true
```

**Notes** — the error trap is the load-bearing part. The codec at this seam reports a fatal error by calling a handler that is expected never to return; the default handler ends the process. Replacing it with one that records the message and unwinds to a saved resume point is what turns "a corrupt screenshot kills the game" into "the screenshot failed". A rebuild in a language with exceptions gets this for free and should simply catch; the only requirement that survives is that a codec failure must not be fatal and must still release the codec's own allocations.

The sink exists because the codec pushes fixed-size blocks and the engine's writers pull. Four kilobytes is an arbitrary block size with no constraint behind it.

The whole file compiles to two logging stubs when the codec is absent. That is the shape a rebuild should keep: screenshots and the handful of loading screens are the only callers, and neither is worth a hard dependency.
