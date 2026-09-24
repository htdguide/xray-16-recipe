# src/xrEngine/tntQAVI.cpp

> Plays a legacy AVI clip as an animated texture, decoding through the host's installed video codecs, with a second clip supplying the alpha channel.

**Needs** — [`tntQAVI.h`](tntQAVI.h.md) · [`xrCore/FS.h`](../xrCore/FS.h.md) · [`device.h`](device.h.md) · [Seam: Audio and video codecs](../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs)
**Used by** — [`tntQAVI.h`](tntQAVI.h.md)
**Tier floor** — T1: it reads a container's chunk tree from memory, hands raw compressed chunks to a decoder at byte offsets, and reinterprets the decoded output as 32-bit pixels.

## Purpose

Some game data ships animated textures as AVI clips, from before the engine adopted a
modern video codec. This plays them: a clip is loaded whole into memory, and each frame the
player is asked for the decompressed image, which is uploaded as a texture.

Two design constraints shape it. First, the clip is an *animated texture*, not a video —
it loops forever, has no audio, and must be samplable at an arbitrary frame at any time.
Second, the codec is whatever the host has installed, because the shipped clips use
codecs of their era.

This is the most platform-bound file in the chapter: it exists only on Windows and only
because the host provides both the container reader and the codec registry. A rebuild
should treat AVI playback as an optional compatibility path and reach for a general
demuxer/decoder instead — see the newer [Theora path](xrTheora_Surface.cpp.md), which is
what the games' actual videos use.

## State

```text
RECORD AviPlayer
  frames        : bytes                  # the whole movie payload, in memory
  index         : list<IndexEntry>       # per frame: offset, length, key-frame flag
  decoder       : codec instance
  output        : bytes                  # one decoded frame, 32-bit, width*height*4 + 4
  in_format     : image description of the compressed stream
  out_format    : image description, forced to 32-bit uncompressed
  width, height : int
  total_frames  : int
  current_frame : int
  nominal_rate  : real                   # frames per second as authored
  play_rate     : real                   # frames per second as currently played
  start_time    : int                    # clock origin, set lazily on first sample
  alpha_clip    : optional<AviPlayer>    # a second clip providing the alpha channel
```

Invariants:

- `current_frame` is initialised to a value **two below the wraparound point**, not to zero
  and not to the "no frame" sentinel. The frame-advance path tests `wanted ==
  current + 1`, and starting at the maximum value would make that test wrap to zero and
  wrongly take the cheap path on the very first frame. Any impossible value two or more
  below the wrap works; this is a guard, not a magic number.
- The alpha clip, when present, has exactly the same dimensions as the main clip.
- The output buffer is four bytes larger than the image needs. The slack is not read; it
  guards against a codec writing past the end, which some of the era's codecs do.

## Loading

**Contract** — reads the whole clip into memory. Returns whether it succeeded; every
failure path closes what it opened and leaves the player unusable rather than
half-initialised. Blocking, and allocates the full compressed size of the clip plus one
uncompressed frame.

```text
FUNCTION load(path)
  IF a file named "<path>_alpha" exists THEN
    alpha_clip = a second player loaded from it

  descend the container to the stream header:  AVI > header list > stream list > stream header
  read the stream header

  read the clip's summary through the host's file interface:
    total_frames, play_rate = rate / scale, width, height
  allocate the output buffer: width * height * 4 + 4

  read the stream format chunk as the decoder's input description
  build the output description: 32-bit, uncompressed, same dimensions
  locate a codec that converts input to output, in fast-decompress mode
  begin decompression

  descend to the movie payload chunk; read it whole into memory
  descend to the index chunk; read it whole into memory
  close the file
```

**Notes** — the file is read twice by two different mechanisms: a low-level chunk walk for
the headers, payload and index, and the host's higher-level clip interface purely to get
the dimensions, length and rate. That duplication is historical. What a rebuild needs is
the *set of facts* extracted — dimensions, frame count, nominal rate, per-frame (offset,
length, is-key-frame), and the compressed and desired uncompressed formats.

The payload and index are loaded whole rather than streamed. These are short texture loops,
so that is affordable and it is what makes arbitrary-frame seeking cheap.

Each frame's compressed data starts **eight bytes past** the offset the index records,
because the index points at the chunk header rather than its payload. That is a container
fact and is frozen.

## `CalcFrame` — which frame is due

**Contract** — maps the global continual clock onto a frame number, looping.

```text
FUNCTION current_frame_number()
  IF start_time is unset THEN start_time = now - 1       # lazily anchor to first use
  RETURN floor((now - start_time) * play_rate / 1000) modulo total_frames
```

**Invariants** — the clock used is the *continual* one, which keeps running while the game
is paused. An animated texture on a screen in the world should not freeze because the
player opened a menu.

The origin is set to one millisecond *before* the first sample rather than to it, so the
elapsed time is never zero on the first call. A zero elapsed time is harmless here, but the
same guard pattern appears throughout the chapter.

## `GetFrame`

**Contract** — hands back the decoded image for the frame that is currently due, and
reports whether it differs from the one handed back last time. The buffer is the player's
own and is valid until the next call. Never fails visibly; a decode failure leaves the
previous image in place.

```text
FUNCTION get_frame() -> (image, changed)
  wanted = current_frame_number()
  IF wanted == current_frame THEN
    RETURN (output, false)                       # nothing to do, and this is the common case
  IF wanted == current_frame + 1 THEN
    current_frame = wanted
    decode(current_frame)                        # sequential: the decoder has the right state
    RETURN (output, true)
  # arbitrary seek
  IF frame `wanted` is not a key frame THEN
    pre_roll(wanted)                             # rebuild decoder state up to wanted-1
  current_frame = wanted
  decode(current_frame)
  RETURN (output, true)
```

**Invariants** — the three cases exist because inter-frame compression makes a frame's
decode depend on the decoder's accumulated state. Same frame: nothing. Next frame: the
state is already right. Any other frame: the state must be reconstructed.

## `PreRoll` — reconstructing decoder state

**Contract** — decodes every frame from a suitable starting point up to but not including
the target, in "preroll, hurry up" mode so the decoder updates its state without producing
output. Not bounds-checked; the caller guarantees the frame number is valid.

```text
FUNCTION pre_roll(target)
  # Walk backwards looking for either a key frame or the frame we already hold
  FOR i FROM target-1 DOWN TO 1
    IF frame i is a key frame THEN
      decode i as (key frame, preroll, hurry up)
      BREAK
    IF i == current_frame THEN
      BREAK                     # our existing state is already frame i's; start from here
  FOR j FROM i+1 TO target-1
    decode j as (not key frame, preroll, hurry up)
```

**Notes** — the "or the frame we already hold" branch is the optimisation that makes
seeking backwards by a few frames cheap: if the last decoded frame is closer than the
previous key frame, resume from it. In a long run of non-key frames this turns an O(gap
from key frame) seek into an O(gap from here) one.

A frame whose recorded length is zero is a *null frame* — an explicit "identical to the
previous" marker — and is flagged as such rather than decoded. The decoder is expected to
advance its state and produce nothing.

One decoder-specific error is accepted as success: "do not draw". A codec of the era
returns it for frames it has consumed but chosen not to render, and treating it as failure
aborts playback on clips that are otherwise fine. That is named in the source as a
workaround for one specific codec, and it is the kind of accommodation a rebuild inherits
along with the data.

## The alpha clip

**Contract** — when a companion clip exists, its decoded frame's *luminance* becomes the
main frame's alpha channel, computed as the plain mean of the three colour channels.

**Notes** — this exists because the container has no alpha and the codecs of the era did
not carry one. Two clips, one carrying colour and one carrying a greyscale mask, is the
workaround the assets were authored with, and a rebuild that wants to load those assets
must reproduce it — including the plain arithmetic mean, which is not a perceptual
luminance and will differ visibly from one.

The alpha clip is driven by the same clock, so the two stay in step without any explicit
synchronisation. They will drift only if their nominal rates differ, which nothing checks.

## `SetSpeed`

**Contract** — sets playback rate as a percentage of the clip's nominal rate, returning the
previous percentage.

**Notes** — the nominal rate is **never assigned**: it is initialised to zero and nothing
writes it, while the *current* rate is set at load from the container. So this divides by
zero on entry and multiplies by zero on exit, and calling it stops playback permanently.
**The intended behaviour is clear and the code is broken**; nothing in the engine calls it.
A rebuild should set the nominal rate at load alongside the current one.

## `GetSize`

**Contract** — hands back the clip's dimensions. Each output is optional.
