# src/xrCore/NET_utils.cpp

> The byte-level packet: a fixed-capacity buffer with a write cursor and a read cursor, plus the quantizations that turn angles, directions and bounded floats into one or two bytes.

**Needs** — [`NET_utils.h`](NET_utils.h.md) · [`_compressed_normal.h`](_compressed_normal.h.md) · [`_matrix.h`](_matrix.h.md) · [`xrstring.h`](xrstring.h.md) · [`client_id.h`](client_id.h.md)
**Used by** — [`net_utils.h`](net_utils.h.md)
**Tier floor** — T1: a wire format read and written by pointing at raw bytes, with exact 32-bit float bit patterns crossing the wire and unaligned reads performed freely.

## Purpose

Every message between a client and a server, and every entity's serialized state inside a save game, is one of these. The packet is deliberately dumb — a flat byte array, a length, and two independent cursors — because the smarts live in the per-entity serializers that call it. What this file owns is the *encoding vocabulary*: the exact byte layout of each primitive, and the quantizations that let a position update cost a handful of bytes instead of a dozen floats.

Both ends of every wire are this same codebase, so the format is **frozen only against itself** (system requirements §5). A rebuild may redesign it wholesale — but must then version-gate it, and must accept that existing save games become unreadable, because a save is a compressed archive of exactly these packets.

## State

```text
RECORD Packet
  data       : bytes[16384]     # fixed capacity; see below
  count      : int              # bytes written so far; the logical length
  read_pos   : int              # independent read cursor
  time_received : int           # when this arrived, filled by the transport
  write_allowed : bool          # guards the text-mirror mode, see Notes
  text_mirror : optional<TextStream>
```

**Invariant** — `count` never reaches the capacity. Every write checks before copying and the check is an assertion, not a returned error: overflowing a packet is a programming error in the serializer, not a runtime condition. Sixteen kilobytes is the cap, chosen to sit under the transport's own fragmentation threshold with room to spare; it is also the reason a packet is a by-value object that callers are careful not to copy casually.

**Invariant** — the read and write cursors are independent and neither bounds the other. A packet is filled, then read from the start, and nothing enforces that ordering; reading past `count` is an assertion in a checked build and undefined otherwise.

**Invariant** — the buffer is packed with no alignment padding anywhere. Multi-byte values are written at whatever offset the cursor happens to be at, so the reader performs unaligned loads by construction. This is one of the places system requirements §4 means when it says unaligned loads are performed.

## Byte layout of the primitives

Every fixed-width value is written in the host's byte order with no conversion — little-endian in practice, everywhere the engine runs. A big-endian port must add a swap at every one of these.

| Written as | Bytes | Notes |
|---|---|---|
| unsigned/signed 8, 16, 32, 64 | 1, 2, 4, 8 | raw, native order |
| float | 4 | the exact bit pattern; no normalization |
| three-component vector | 12 | three floats, x then y then z |
| four-component vector | 16 | four floats |
| transform | 48 | four three-component vectors: the three basis rows, then the translation. The fourth column is **not** transmitted |
| client identifier | 4 | one unsigned 32-bit word |
| string | length + 1 | the bytes followed by one zero byte; a null string writes the zero byte alone |

The transform's omission is load-bearing: the reader reconstructs the missing column as `(0, 0, 0, 1)`, which is correct for every rigid transform the engine sends and wrong for anything with projection in it. Nothing sends a projective transform over the wire, and a rebuild should keep the saving rather than "fix" it.

## Quantized encodings

These are the reason the format is cheap, and they are the part a rebuild is most likely to get subtly wrong.

```text
FUNCTION write_bounded_16(a: real, min: real, max: real)
  # a must lie within [min, max]; the caller guarantees it
  q <- (a - min) / (max - min)
  write unsigned 16-bit: floor(q * 65535 + 0.5)

FUNCTION read_bounded_16(min, max) -> real
  v <- read unsigned 16-bit
  RETURN (v * (max - min)) / 65535 + min

FUNCTION write_bounded_8(a: real, min: real, max: real)
  q <- (a - min) / (max - min)
  write unsigned 8-bit: floor(q * 255 + 0.5)

FUNCTION read_bounded_8(min, max) -> real
  v <- read unsigned 8-bit
  RETURN (v / 255.0001) * (max - min) + min
```

**Notes** — the eight-bit round trip is deliberately asymmetric: the writer divides the range into 255 steps but the reader divides by `255.0001`. That fraction pulls the reconstructed maximum a hair *below* `max`, so that a value written at the top of the range reads back strictly inside the range and the reader's own bounds assertion cannot fire on a floating-point rounding. The sixteen-bit reader has no such fudge and instead widens its assertion by an epsilon at both ends. Both are workarounds for the same problem and a rebuild only needs one of them — but if it keeps the wire format it must keep the *writer's* arithmetic exactly, because that is what is on the wire.

An angle is a bounded float over `[0, 2π)` after being normalized into that interval, in either width. Sixteen bits gives about a hundredth of a degree; eight gives about one and a half degrees, which is enough for a torso yaw and not enough for an aim direction.

A **direction** is a unit vector packed into sixteen bits by the octahedral-style scheme in [`_compressed_normal.h`](_compressed_normal.h.md) — three sign bits and a point on a triangular grid. A **scaled direction** is that same sixteen-bit direction followed by a full float magnitude: eighteen bytes' worth of information in six. A zero-length vector is sent as the unit vector `(0, 0, 1)` with magnitude zero, so that the receiver reconstructs a zero vector rather than a normalized garbage direction; the sender's threshold for "zero" is the engine's small epsilon.

## Back-patched chunks

**Contract** — a serializer that does not know a sub-record's length in advance opens a chunk, writes the record, and closes it; closing computes the length and writes it back over the placeholder. Two widths exist, one byte and two, and the choice is the serializer's own — an over-long chunk trips an assertion rather than promoting itself.

```text
FUNCTION open_chunk(width) -> position
  position <- write cursor
  write a zero of the given width          # placeholder
  RETURN position

FUNCTION close_chunk(position, width)
  size <- write cursor - position - width  # the length EXCLUDES the length field
  assert size fits in width
  overwrite the `width` bytes at `position` with size
```

**Invariant** — the recorded length excludes the length field itself, so a reader skips a chunk by advancing `width + size` from the field's position. Getting that convention backwards desynchronizes every subsequent field in the packet, silently.

## String reading

**Contract** — reading a string scans forward from the read cursor to the first zero byte, copies the bytes (including the terminator) out, and advances past it. Three destinations exist — a caller's fixed buffer, a growable string, and an interned string — and they differ only in where the bytes land. A fourth, bounded form takes the destination's capacity and asserts rather than overrunning it; the unbounded form into a fixed buffer does not, and is the one real hazard in this file. Skipping a string advances the cursor without copying.

**Notes** — nothing bounds the scan by `count`. A packet whose last string is unterminated walks off the end of the buffer. The buffer is a fixed array inside the packet object, so the walk stays inside the object's own memory and then into whatever follows it — a rebuild should bound the scan at the packet length and treat a missing terminator as a malformed packet, since packets arrive from the network and are therefore hostile input.

## The text mirror

**Contract** — a packet may be attached to a text stream, in which case every *write* is additionally emitted to it as a named key/value line, and reads are served *from* it instead of from the byte buffer. When attached, the raw byte-level operations — seek, tell, advance, end-of-buffer, and the untyped read — are not available and assert if used.

**Notes** — this is a debugging and tooling facility: it lets an entity's serialized state be dumped as readable configuration and re-read from an edited version, which is how the tools inspect a save game without a bespoke parser. It is also the source of most of the branching in this file: nearly every typed accessor is "if a mirror is attached, delegate to it; otherwise touch the bytes".

A rebuild should express this as a *serializer interface* with two implementations — binary and textual — rather than as a flag inside the binary one. The current shape leaks: writing a null string has to detach the mirror, write the terminator byte, and reattach, because the mirror's own string writer would emit an empty line instead. That contortion is a symptom, not a design.

The guard object that flips the write-permission flag around each typed write exists so that the untyped write can refuse to run while a mirror is attached *except* when it is being called from a typed write that has already mirrored the value. It is a lock in all but name, and it disappears entirely under the two-implementation shape.
