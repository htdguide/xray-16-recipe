# src/xrSound/SoundRender_Source.cpp

> One sound asset: how it is found, how its sidecar is read, and how compressed audio becomes PCM
> at an arbitrary byte offset.

**Needs** — [`SoundRender_Source.h`](SoundRender_Source.h.md) · [`SoundRender_Core.h`](SoundRender_Core.h.md) · [`Sound.h`](Sound.h.md) · [Seam: Audio and video codecs](../../SYSTEM-REQUIREMENTS.md#seam-audio-and-video-codecs)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it parses a frozen binary sidecar out of a metadata field and converts between
byte offsets and sample offsets in the decoder's own units.

## Purpose

A source is an asset's *description* plus the ability to produce PCM from it. It holds no audio:
every emitter opens its own decoder over the same file and seeks independently, so a hundred
footsteps share one description and cost a hundred file handles, not a hundred copies of the wave.

The other half of this file is the sidecar — the engine-specific metadata every shipped sound
carries, including the attributes the AI's hearing sense reads.

## State

```text
RECORD Source
  physical_path : text     # the resolved file, with extension
  logical_name  : text     # the cache key: no extension, lowercased
  length_sec    : real
  bytes_total   : int      # of *decoded* PCM at the chosen output format
  data_info     : SoundDataInfo
  info          : SoundSourceInfo

RECORD SoundDataInfo
  format            : ENUM { Unknown, PCM16, Float32 }
  channels          : int
  samples_per_sec   : int
  avg_bytes_per_sec : int    # = samples_per_sec × block_align
  block_align       : int    # = bits_per_sample / 8 × channels ; one frame of all channels
  bits_per_sample   : int
  bytes_per_buffer  : int    # = BLOCK_MS × avg_bytes_per_sec / 1000

RECORD SoundSourceInfo          # the sidecar
  base_volume     : real = 1.0
  min_distance    : real = 1.0    # inside this radius the source is at full volume
  max_distance    : real = 300    # beyond this it is not heard at all
  max_ai_distance : real = 300    # how far it carries *to a creature*
  game_type       : int  = 0      # AI event classification, a bit set
```

Invariants:

- `bytes_total` is in the *output* format's units, not the file's. Every cursor, stop time and
  length in the chapter derives from it, so choosing float output changes every byte count but no
  timing.
- `block_align` is the atom of every seek: a byte offset is only meaningful when it is a whole
  number of frames.
- `max_distance` and `max_ai_distance` must both be at least 0.1; an asset failing that is rejected
  at load, because a zero range would make the distance attenuation divide toward infinity.

## The sidecar — what pairs with every sound asset

Every shipped sound carries engine metadata inside the audio container's own comment field: the
**first user comment** is not text but a small binary record. This is how a 2007 tool chain attached
engine data to a standard audio file without a second file on disk, and it is frozen — the shipped
sound bank contains tens of thousands of these.

```text
RECORD Sidecar                     # little-endian, packed, no padding
  version : int (32-bit)
  # version 1
  min_distance    : real (32-bit)
  max_distance    : real (32-bit)
  game_type       : int  (32-bit)
  # version 2 inserts base_volume between max_distance and game_type
  base_volume     : real (32-bit)
  # version 3 appends
  max_ai_distance : real (32-bit)
```

Read as: version 1 is (min, max, type) with base volume implied 1.0 and the AI range equal to the
audible range. Version 2 adds an authored base volume. Version 3 — the current one — separates the
AI range from the audible range, which is the whole point of the format's last revision: **how far a
sound carries to a creature is a design decision independent of how far the player hears it.** A
silenced pistol is audible to the player at thirty metres and perceptible to an NPC at five; a
distant siren is the other way round.

An unknown version, or no comment at all, leaves every field at its default and logs outside
shipping builds. That is deliberate: a modder's freshly encoded file plays at default range rather
than failing to load.

**`game_type`** is the other half of the AI story. It is a bit set, not an enumeration: the high
bits classify the *actor* (weapon, item, monster, anomaly, world) and the low bits the *event*
(shooting, stepping, dying, talking, eating, breaking, exploding, ambient…), and a real value is
the union of one of each — "a weapon shooting", "a monster stepping". The creature's hearing sense
reacts differently to each combination, so this single word is what turns an audible sound into a
meaningful stimulus. The vocabulary is defined with the AI, not here; see
[`xrServerEntities/ai_sounds.h`](../xrServerEntities/ai_sounds.h.md). Two of its values are known
to this module: world ambience, which is exempt from occlusion, and "no sound", which suppresses AI
announcement entirely.

## `load`

**Contract** — Resolves a logical sound name to a file and reads its description. Returns whether it
succeeded; a failure is not fatal anywhere in the engine. Blocks on file I/O. Does not decode audio.

```text
FUNCTION load(name) -> bool
  logical_name ← strip_extension(name)          # lowercased on case-folding platforms
  candidate ← logical_name + ".ogg"
  # A level may override a global sound with its own copy of the same name.
  IF NOT exists_under(level_root, candidate) THEN
    candidate ← resolve_under(global_sounds_root, candidate)
  IF NOT exists(candidate) THEN
    log a warning; in the editor substitute a placeholder asset
    RETURN false
  RETURN read_description(candidate)
```

**Notes** — The level root is searched before the global sound root, which is how a level ships a
variant of a shared sound without renaming every reference to it. The extension is appended, never
taken from the caller — the game data references sounds with and without a stale `.wav` extension
and both must resolve to the same `.ogg`.

## `read_description`

**Contract** — Opens the file through the decoder, reads its format and length, chooses the output
PCM format, parses the sidecar, and validates the ranges. Leaves no decoder open.

```text
FUNCTION read_description(path) -> bool
  open the file and attach a decoder
  samples_per_sec ← decoder.rate ; channels ← decoder.channels

  # Float output when both the device offers it and the player allows it;
  # otherwise 16-bit. This choice is global and set before any asset loads.
  IF device supports float PCM THEN format ← Float32, bits ← 32
  ELSE                              format ← PCM16,   bits ← 16

  block_align       ← bits / 8 × channels
  avg_bytes_per_sec ← samples_per_sec × block_align
  bytes_per_buffer  ← BLOCK_MS × avg_bytes_per_sec / 1000

  bytes_total ← decoder.total_samples × block_align
  length_sec  ← bytes_total / avg_bytes_per_sec

  parse the sidecar from the first comment, by version (above)
  IF max_distance < 0.1 OR max_ai_distance < 0.1 THEN RETURN false
  close the decoder
  RETURN true
```

**Invariants** — `length_sec` is derived from the decoded byte count rather than asked of the
decoder, so that the emitter's clock and its cursor cannot disagree about where the end is.

## `open` / `close`

**Contract** — `open` gives one emitter its own decoder over the asset, reading through the virtual
filesystem rather than the operating system's file API — sounds live inside compressed archives.
Asserts if the file is missing or empty; by this point the description loaded, so a missing file is
a corrupt installation, not a content error. `close` releases it. Each emitter holds at most one
decoder, and a demoted emitter closes it.

**Notes** — The decoder is attached through four callbacks — read, seek, tell, close — over the
engine's reader. The read callback returns whole units and clamps to what remains, because the
underlying reader is a window into an archive and reading past its end is not an error there, just
wrong. The seek callback must support absolute, relative and from-the-end addressing, because the
decoder seeks backwards to find stream boundaries. This callback shape is exactly the "open from
callbacks" surface the codec seam promises.

## `decompress`

**Contract** — Produces exactly `size` bytes of PCM at an absolute byte offset within this asset,
into a caller-supplied buffer, using a caller-supplied decoder. Runs on the streaming worker. Seeks
only when the decoder is not already at the requested position, which is the common case for
sequential streaming and makes the seek free.

```text
FUNCTION decompress(dest, byte_offset, size, decoder)
  sample_offset ← byte_offset / block_align      # byte offsets are frame-aligned
  IF decoder.position ≠ sample_offset THEN decoder.seek(sample_offset)
  IF format IS Float32 THEN decode_float(decoder, dest, size)
  ELSE                      decode_integer(decoder, dest, size)
```

Both decode loops keep reading until the request is satisfied or the decoder reports an
unrecoverable condition. **The distinction between recoverable and fatal is load-bearing**: a hole
in the bitstream or an undecipherable link is *informational* — the decoder resynchronizes and the
loop continues, because the shipped sound bank contains files with minor corruption and the game
must not stop for them. A read error, a bad header, a wrong version or an unseekable stream ends
the loop and leaves the rest of the block as whatever it held.

The float path additionally **clamps every sample to [-1, 1]**. Decoders may legitimately emit
values slightly outside that range, and the mixer does not clamp; without this a loud asset
produces audible wrapping. The integer path needs no clamp because the decoder saturates for it.

The float path also interleaves — the decoder hands back one plane per channel and the device wants
frames — while the integer path receives interleaved data directly. That asymmetry is the decoder's
API, not a decision.

## `unload`

**Contract** — Zeroes the length and byte count. Called before a reload during a content refresh.
There is nothing else to release: a source holds no audio.
