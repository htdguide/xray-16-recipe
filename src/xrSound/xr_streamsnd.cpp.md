# src/xrSound/xr_streamsnd.cpp

> Dead code: the pre-Vorbis music streamer, built on a Windows-only compressed-audio path and a
> circular device buffer. Not compiled.

**Needs** — [`xr_streamsnd.h`](xr_streamsnd.h.md) · [`SoundRender_Core.h`](SoundRender_Core.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1 as written: it locks a device buffer and writes PCM into two wrapped spans.

## Purpose

An earlier generation's music playback, excluded from the build and retained only as history. Its
job is now done by an ordinary looped 2D emitter over a Vorbis asset — music is not a special case
in the shipped engine, it is a sound whose type selects the music volume slider.

**A rebuild should not implement this file.** It is documented because the mirror must be complete,
and because one of its ideas is worth knowing.

## The idea worth keeping

This streamer used a **single circular device buffer** with a write cursor chasing the device's read
cursor, rather than a queue of discrete buffers. Each update it measured the gap between the two
cursors — wrapping the subtraction around the buffer's length — and decoded another chunk when the
gap exceeded the chunk size. End of stream was detected by the read cursor passing the write cursor.

That is the other classic streaming shape, and it is not worse than the queue the current code uses;
it trades a wrap-aware cursor comparison for not having to manage buffer identities. The current
design won because the queue model is what the audio seam offers portably.

The buffer was 88 KB with a 44 KB decode chunk — a deliberate two-to-one ratio so that a decode
always fitted in the free half.

## What else is here, and why it is not worth transcribing

Decoding went through a Windows-only system codec manager: it opened a conversion stream from the
file's compressed format to plain PCM, asked the system to *suggest* the destination format, and
ran fixed-size conversions through it. The container was RIFF, walked chunk by chunk for the format
header and the data offset. Looping was a count that the update decremented at end of stream and
replayed until exhausted, with a sentinel meaning forever.

All of that is the *problem* "decode a compressed stream incrementally and keep a device fed",
which the current chapter solves portably. The container, the codec manager and the buffer locking
are ecosystem artefacts with no successor.

## Notes

The file does not compile as written even on its own platform — several calls have malformed casts
and the non-Windows path leaves whole functions with no body. It has rotted since being excluded.
Do not treat any detail in it as a specification.
