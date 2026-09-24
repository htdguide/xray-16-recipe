# src/xrNetServer/NET_Compressor.cpp

> Compresses an envelope's payload if and only if that makes it smaller, tags the result so
> the receiver knows which happened, and checksums it so a corrupt payload is caught before it
> is expanded.

**Needs** — [`NET_Compressor.h`](NET_Compressor.h.md) · [`NET_Common.h`](NET_Common.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression) · [Seam: Threads](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — reached through its declarations in [`NET_Compressor.h`](NET_Compressor.h.md); callers name that, not this file.
**Tier floor** — T1: it writes a tag, a checksum and a compressed body into a caller-owned
span at fixed offsets.

## Purpose

Compression here is *opportunistic*, and that word carries the whole design. Game updates are
short and already dense — quantized angles and 16-bit identifiers do not compress well — so a
compressor applied blindly would often grow the payload. This file decides per envelope, and
records which way it went in one byte so the receiver need not guess.

## State

```text
RECORD CompressorStats            # diagnostic only, gathered when a console variable asks
  total_in   : int
  total_out  : int
  per_size   : map<int, SizeBucket>     # keyed by exact input length

RECORD SizeBucket
  attempts          : int
  unlucky_attempts  : int    # times compression was tried and did not shrink the payload
  total_output      : int

CONSTANT compress_threshold = 36   # inputs this small or smaller are never even attempted
```

**Invariants** — the compressed form is only ever emitted when it is *strictly* smaller than
the verbatim form including its own one-byte tag. Ties go to verbatim, which keeps the decode
path cheaper for no size cost.

## The payload layout

This is the inner envelope — it sits inside the outer one described in
[`NET_Common.cpp`](NET_Common.cpp.md), which has already written its own three-byte header in
front.

```text
COMPRESSED form
  tag       : int (8-bit)     # 0xC1
  checksum  : int (32-bit)    # CRC32 of the body that follows
  body      : bytes           # compressed

VERBATIM form
  tag       : int (8-bit)     # 0xC0
  body      : bytes           # the input, copied
```

The verbatim form carries no checksum. That asymmetry is deliberate in effect if not
necessarily in intent: a corrupt *compressed* body can expand into anything and must be
rejected, while a corrupt verbatim body is merely a corrupt message, which the layers above
already tolerate.

## `Compress`

**Contract** — writes the tagged payload into the caller's destination span, which the caller
sized with `worst_case_size`. Returns bytes written. Never fails: if compression does not help
or is disabled, the input is copied. Serialized by the compressor's lock only around the
compression call itself, because the underlying compressor keeps working state.

```text
FUNCTION compress(dest, dest_capacity, src, n) -> int
  worth_trying = (n > 36)
  offset = 5                      # one tag byte plus four checksum bytes

  IF worth_trying AND compression enabled AND NOT direct_connect THEN
    LOCK compressor DURING
      out = offset + encode(dest + offset, dest_capacity - offset, src, n)
  ELSE
    out = n                       # sentinel: "no attempt made"

  IF out < n THEN
    dest.tag      = payload_compressed
    dest.checksum = crc32(dest + offset, out)
    RETURN out
  ELSE
    dest.tag = payload_uncompressed
    copy src into dest + 1
    RETURN n + 1
```

**Notes** — three independent conditions suppress the attempt, and each says something
different. **Direct-connect** means client and server share a process, so the bytes never
leave it and compressing them is pure loss. The **enabled** flag defaults to *off*, which is
the surprising one: the shipping default is to send envelopes verbatim, and compression is a
console variable an operator turns on. **The 36-byte threshold** is the one number here with
no discoverable derivation.

There is a defect in the checksum line, and it is worth stating because it tells you the
compressed path is not exercised. The length handed to the checksum is the *total* payload
length including the five-byte prefix, not the body length, so both the writer and the
verifier checksum the body **plus five bytes of whatever follows it** — the sender's scratch
buffer past the compressed body, the receiver's datagram buffer past the end of the datagram.
Those five bytes are not the same on the two ends, so a genuinely compressed envelope would
fail verification and take the process down with it. It does not happen because compression
defaults to off. A rebuild should checksum exactly the body, and should then actually turn
compression on and watch it work.

## `Decompress`

**Contract** — reads the tagged payload and writes the original bytes into the caller's
destination span. Returns bytes written. A checksum mismatch is fatal, not recoverable: there
is no path that drops the datagram and continues.

```text
FUNCTION decompress(dest, dest_capacity, src, n) -> int
  IF src.tag is not payload_compressed THEN
    copy n - 1 bytes from src + 1 into dest        # verbatim form
    RETURN n - 1

  REQUIRE crc32(src + 5, n) = src.checksum         # mismatch is fatal
  LOCK compressor DURING
    RETURN decode(dest, dest_capacity, src + 5, n - 5)
```

**Notes** — treating a bad checksum as fatal is a decision a rebuild should reconsider. On the
wire a corrupt datagram is a normal event, and the correct response is to drop it and count
it; terminating the process hands any peer that can flip a bit a way to stop the game.

An empty payload returns zero rather than reading the tag, so a zero-length datagram is
harmless.

## `worst_case_size`

**Contract** — returns the largest possible compressed output for an input of `n` bytes, plus
the framing. Callers size their destination with it before compressing, so it must be an upper
bound; returning an estimate would turn a pathological input into an overrun.

## `DumpStats`

**Contract** — prints the totals and, unless asked to be brief, one line per distinct envelope
size: how many envelopes of that size were seen, how many failed to shrink, and the mean
output. Diagnostic; no effect on traffic.

**Notes** — the statistics are only gathered when a console variable is set, and `attempts`
counts only envelopes over the threshold, so the "unlucky" ratio describes the compressor's
success on the payloads it was actually offered rather than on all traffic. That is the useful
reading, but it is not the obvious one.
