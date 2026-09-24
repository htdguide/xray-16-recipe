# src/xrEngine/xrTheora_Surface.cpp

> A playable video clip — one or two Theora tracks, the playback clock, looping, and the conversion of a decoded frame into something the graphics device can sample.

**Needs** — [`xrTheora_Surface.h`](xrTheora_Surface.h.md) · [`xrTheora_Stream.h`](xrTheora_Stream.cpp.md) · [`xrCore/FS.h`](../xrCore/FS.h.md) · [Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device) · [Seam: Audio and video codecs](../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs)
**Used by** — [`xrTheora_Surface.h`](xrTheora_Surface.h.md)
**Tier floor** — T1: the inner loop writes packed 32-bit pixels into a mapped texture at a caller-supplied stride, reading three planes at three different strides.

## Purpose

A video in this engine is always a texture: the intro, the loading screens and the
in-world monitors are all surfaces the renderer samples. This file is the bridge from
"a decoded YUV frame" to "a texture the device can sample", plus the playback state — the
clock, looping, pause, and the two-track colour-and-alpha arrangement.

The interesting decision is the *two* conversion paths. When the graphics device can do the
colour-space conversion in a shader, the frame is uploaded almost verbatim and the
conversion is free; when it cannot, the conversion is done on the processor, per pixel.

## State

```text
RECORD VideoClip
  colour     : optional<TheoraTrack>
  alpha      : optional<TheoraTrack>     # a separate file, same dimensions and duration
  start_time : int                       # clock value playback was anchored at
  play_time  : int                       # milliseconds into the clip
  total_time : int
  ready      : bool
  shader_convert : bool                  # the device converts YUV in a shader
  prefetch   : int                       # counts up from -2; see below
  playing    : bool
  looped     : bool
```

Invariants:

- The alpha track, when present, has the **same duration, dimensions and pixel format** as
  the colour track. Nothing enforces this outside debug builds, and a mismatch corrupts
  memory rather than misplaying.
- `total_time` is never zero for a loaded clip; a zero-length clip would divide by zero in
  the loop wrap.
- The pixel format is assumed to be 4:2:0 throughout — the chroma planes at half resolution
  in both axes. The source says so explicitly: the game's videos were produced by one
  encoder and that is what it emits. A rebuild must either assume it too or handle the
  other Theora formats.

## The prefetch counter

**Contract** — playback does not start on the frame `Play` is called. The counter begins at
-2 and increments once per update; only when it reaches zero is the clock anchored and
`play_time` allowed to advance.

**Notes** — this discards the *first two* updates. The first update is where the file's
initial pages are read and the first frames decoded, which is slow; anchoring the clock
before that work means the clip starts two hundred milliseconds in. Two frames of grace,
then anchor. The count is a hand-tuned guard, not a derived value, and a rebuild that
decodes ahead on a worker does not need it.

## `Update`

**Contract** — advances the clip to a given clock value and reports whether the image
changed. Requires a loaded clip. Stops a non-looping clip that has run out, returning false
that last time.

```text
FUNCTION update(now) -> changed
  IF prefetch < 0 THEN
    prefetch = prefetch + 1
    IF prefetch == 0 THEN start_time = now      # anchor after the slow first updates
    play_time = 0
  ELSE IF playing THEN
    play_time = now - start_time

  IF not playing THEN RETURN false

  IF play_time >= total_time THEN
    IF looped THEN
      start_time = start_time + total_time      # advance the anchor, don't re-read the clock
      reset both tracks
    ELSE
      stop; RETURN false

  changed = colour.decode(play_time)
  IF alpha THEN changed = changed OR alpha.decode(play_time)
  RETURN changed
```

**Invariants** — looping advances the *anchor* by exactly one clip length rather than
re-anchoring to the current clock. That keeps the loop free of accumulated drift: a clip
played for an hour is at exactly the same phase as one played from the start, and a frame
that overran does not shorten the next pass.

Both tracks are decoded to the same playback time, which is what keeps them in step.

## `Load`

**Contract** — loads the colour track from a path, then looks for a companion alpha track
at the same path with `#alpha` inserted before the extension, and loads it if present.
Queries the graphics device for shader-based colour conversion. Returns whether the clip is
usable; on failure both tracks are released.

**Notes** — the alpha companion naming is a data convention and is frozen by the shipped
files: same directory, same base name with `#alpha` appended before the extension.

The original once required power-of-two dimensions and no longer does — the assertion is
commented out — because the texture path now pads instead. See the size query below.

## `Width` / `Height`

**Contract** — each returns either the clip's true size or the next power of two at or
above it, by a flag.

**Notes** — this is the padding the size assertion was dropped for. A device that requires
power-of-two textures gets a padded texture with the video in its corner, and the caller
compensates in the texture coordinates. A rebuild targeting any modern device asks for the
real size only.

## `DecompressFrame` — the conversion

**Contract** — writes the current frame into a caller-supplied 32-bit pixel buffer whose
rows are `video_width + padding` wide, and reports how far it wrote. The buffer is normally
a mapped texture. Reads the colour track's planes and, if present, the alpha track's
luminance plane.

### Path one: the processor converts

```text
FOR EACH row h
  Y row  = luma plane  + luma stride * h
  UV row = chroma planes + chroma stride * (h / 2)     # 4:2:0: one chroma row per two luma
  FOR EACH column w
    y = Y[w]; u = U[w/2]; v = V[w/2]
    C = y - 16; D = u - 128; E = v - 128               # remove the studio-range offsets
    R = clamp((298*C           + 409*E + 128) >> 8, 0, 255)
    G = clamp((298*C - 100*D   - 208*E + 128) >> 8, 0, 255)
    B = clamp((298*C + 516*D           + 128) >> 8, 0, 255)
    write opaque (R, G, B)
  skip the row padding
```

**Invariants** — this is the standard studio-range BT.601 conversion in 8.8 fixed point.
The offsets 16 and 128 are the studio-range black level and the chroma zero point, both
frozen by the encoded data. The coefficients 298, 409, 100, 208 and 516 are the BT.601
matrix scaled by 256; the added 128 is the rounding half. A rebuild may use floating point
and get the same picture, but **must not** assume full-range YUV — the black level would
be wrong and the image washed out.

### Path two: the device converts

```text
FOR EACH pair of rows, and each pair of columns within them
  read four luma samples and the one chroma pair they share
  write four pixels, each packed as:
      alpha = 255, red = luma, green = u, blue = v
```

**Notes** — the frame is uploaded with the *components in the wrong channels on purpose*:
luma in red, and the two chroma samples replicated into green and blue across all four
pixels of the 2×2 block. The shader unpacks them. That makes the processor's job a
rearrangement instead of a matrix multiply, and the chroma upsampling becomes the device's
bilinear filter rather than the nearest-neighbour replication path one does.

The two paths therefore do not produce identical images: path two has smoother chroma. That
is accepted.

### The alpha overlay

```text
FOR EACH row, FOR EACH column
  a = (luma - 16) / 0.858823                   # studio range expanded to full range
  replace the destination pixel's alpha with clamp(a)
```

**Invariants** — the divisor is the sum of the three BT.601 luma coefficients
(0.256788 + 0.504129 + 0.097906), which is the luma value a full-white pixel produces in
studio range. Dividing by it maps studio-range luma back to a full 0..255 alpha, so an
alpha track's white is fully opaque.

**Notes** — the alpha pass indexes the destination with a **pre-increment**, so it writes
starting at the second pixel and runs one past the end of the last row, and it does not
skip the row padding the colour pass accounts for. The alpha channel is therefore offset by
one pixel and progressively skewed. This is a live bug in the original; a rebuild should
index the same way the colour pass does.

## `Play` / `Pause` / `Stop` / `Reset` / `IsPlaying` / `Valid`

**Contract** — `Play` anchors the clock, sets the loop flag and re-arms the prefetch
counter. `Pause` toggles the playing flag without touching the clock, so a paused clip
resumes *at the wall-clock position it would have reached* — it does not freeze. `Stop`
halts and rewinds both tracks. `Reset` rewinds without changing the playing state.

**Notes** — the pause behaviour is a consequence of the clock being external and is worth
flagging: a rebuild that wants a true pause must record the elapsed time at pause and shift
the anchor on resume.

## The standalone display path

**Contract** — an optional build mode that opens its own window and blits the YUV planes to
a hardware overlay, bypassing the engine's renderer entirely.

**Notes** — a development harness for testing video decode without the engine. It crops to
the encoded frame rectangle using the track's declared offsets, and it swaps the two chroma
planes because the overlay format orders them the other way. Both are real facts about the
data; neither is reachable in a shipping build. A rebuild should drop this and test through
the renderer.
