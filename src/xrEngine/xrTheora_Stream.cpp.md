# src/xrEngine/xrTheora_Stream.cpp

> One Theora video track inside an Ogg container — headers, a frame index derived by scanning, and decode-to-a-requested-time with key-frame pre-roll.

**Needs** — [`xrTheora_Stream.h`](xrTheora_Stream.h.md) · [`xrCore/stream_reader.h`](../xrCore/stream_reader.h.md) · [`xrCore/FS.h`](../xrCore/FS.h.md) · [Seam: Audio and video codecs](../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs)
**Used by** — [`xrTheora_Stream.h`](xrTheora_Stream.h.md) · [`xrTheora_Surface.cpp`](xrTheora_Surface.cpp.md)
**Tier floor** — T1: it feeds a decoder library through a buffer the library hands out, and receives back plane pointers with strides it must not copy.

## Purpose

Videos ship as Theora in an Ogg container: the intro, the loading screens, and the
in-world screens. This file owns *one track* of one such file and answers exactly one
question: given a playback position in milliseconds, produce the frame that should be on
screen.

The layering matters. This is the *track*; [`xrTheora_Surface.cpp`](xrTheora_Surface.cpp.md)
is the *clip*, which may be two tracks (colour and alpha) in two separate files and owns
the playback clock.

## State

```text
RECORD TheoraTrack
  source         : stream reader over the file
  sync, stream   : container demuxer state
  info, comment  : the track's declared parameters
  decoder        : decoder state
  frame          : YUV plane set (pointers into the decoder's own memory, plus strides)
  decoded_frame  : int       # index of the last frame decoded; -1 before any
  total_ms       : int       # the track's duration
  key_rate       : int       # frames between key frames; constant for this format
  frames_per_ms  : real
```

Invariants:

- `key_rate` is **constant across the whole track**. Theora as produced by the encoder the
  game data was made with emits key frames at a fixed interval, and the entire seek strategy
  depends on it — the file says so explicitly. A track with irregular key frames would seek
  to the wrong place, silently. A rebuild that accepts arbitrary Theora must find the
  preceding key frame by search instead of by arithmetic.
- `decoded_frame` only ever moves forward within a playback pass. Seeking backwards is done
  by resetting the whole track and replaying.
- The frame's plane pointers are owned by the decoder and are valid until the next decode.

## Reading

**Contract** — refills the demuxer's buffer from the file in fixed 4096-byte bites, bounded
by what remains. Returns how many bytes were added; zero means end of file.

**Notes** — the buffer is requested *from* the demuxer rather than allocated, because the
demuxer needs the data contiguous with what it already holds. That is the library's
contract and survives into any rebuild that uses a streaming demuxer.

## `ParseHeaders`

**Contract** — finds the Theora track among the container's tracks, reads its three header
packets, initialises the decoder, and then — the expensive part — **scans the entire file
to count frames and measure the key-frame interval**. Blocking; reads the whole file once.
Returns whether a usable Theora track was found.

```text
FUNCTION parse_headers()
  # Phase 1: find the track
  WHILE more data
    FOR EACH complete page
      IF the page is not a beginning-of-stream page THEN
        hand it to the chosen track; stop looking
      open a trial track for this page's serial number
      IF its first packet parses as a Theora header AND we have no track yet THEN
        adopt this track
      ELSE discard the trial track          # some other codec: audio, or a second video
  IF no track adopted THEN RETURN false

  # Phase 2: two more header packets
  WHILE fewer than three headers read
    pull packets from the track and parse each as a header
    refill from the file when the track runs dry
  IF fewer than three THEN RETURN false

  initialise the decoder from the parsed parameters
  frames_per_ms = (fps numerator / fps denominator) / 1000

  # Phase 3: measure the track by walking every packet
  count = 0; previous_key = 0
  WHILE not end of file
    FOR EACH packet
      IF key_rate is still unknown AND this packet is a key frame THEN
        key_rate = count - previous_key; previous_key = count
      count = count + 1
    refill; feed pages to the track
  total_ms = floor(count / frames_per_ms)
  reset to the beginning
  RETURN true
```

**Notes** — phase 3 is marked in the source as a hack and it is: it reads the whole file
from disk purely to learn the frame count and the key-frame interval. The container
*records* a duration in its final page's granule position, and a rebuild should read it
from there.

The interval is measured from the gap between the **first two key frames** and then never
re-measured, which is the concrete form of the constant-key-rate assumption. If the first
frame is a key frame and the second is too, the interval measures as one and every seek
becomes a full re-decode — correct but slow.

Failure during header parsing terminates the process rather than returning. That is
unacceptable in a rebuild; the caller has a perfectly good failure path and a corrupt video
should not kill a session.

## `Decode`

**Contract** — advances the track so that the frame for a given playback time is the
current one. Returns whether the current frame changed. Requires the time to be within the
track. Seeks only forwards; a backwards request is satisfied by the caller resetting first.

```text
FUNCTION decode(time_ms) -> changed
  target = floor(time_ms * frames_per_ms)
  key    = target - (target modulo key_rate)     # the key frame the target depends on
  IF decoded_frame >= target THEN RETURN false

  WHILE decoded_frame < target
    FOR EACH available packet that is not a header
      decoded_frame = decoded_frame + 1
      IF decoded_frame < key THEN CONTINUE      # skip entirely: not needed for the target
      feed the packet to the decoder
      IF decoded_frame >= target THEN done
    IF not done THEN refill from the file and feed pages to the track

  pull the YUV planes out of the decoder
  RETURN true
```

**Invariants and decisions:**

- **Frames before the target's key frame are skipped without being decoded at all.** Not
  decoded-and-discarded: the packet is counted and dropped. This is only sound because the
  decoder is reset to the key frame's state by the key frame itself, which by definition
  depends on nothing earlier. It makes a forward seek cost (distance from the key frame)
  rather than (distance from the current position).
- Frames between the key frame and the target *are* decoded, because the target depends on
  them. There is no cheaper option with inter-frame compression.
- The scan is driven by the frame index, not by the decoder's own timestamps. That is what
  makes the fixed-key-rate assumption load-bearing: the index-to-key-frame arithmetic is
  pure division.

## `Reset`

**Contract** — rewinds the file, resets the demuxer and the track, and marks no frame
decoded. This is the only way to move backwards.

## `Load`

**Contract** — opens the file through the virtual filesystem as a stream reader, parses the
headers, and rewinds. Returns whether it succeeded.

**Notes** — the reader is a *stream* rather than a whole-file buffer, because videos are
large and the archive layer can decompress them incrementally. This is one of the few
places in the engine that genuinely needs streaming rather than a mapped or fully-read
file.
