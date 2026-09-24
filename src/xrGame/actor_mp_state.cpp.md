# src/xrGame/actor_mp_state.cpp

> The bit-packed wire form of a networked player's state: what is sent, at what precision, and in what order.

**Needs** — [`actor_mp_state.h`](actor_mp_state.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`actor_mp_state.h`](actor_mp_state.h.md)
**Tier floor** — T1: a bit-exact wire layout with relied-upon quantization

## Purpose

A player's state is sent to every other client several times a second, so the format is
optimized to the bit. This file is that format: it decides which quantities are worth full
precision, which survive being crushed into a handful of bits, and how the small integer
fields are packed together so that they cost bytes rather than fields.

Both ends of the wire are this codebase, so the format is frozen only against itself — a
rebuild may redesign it, and must then version-gate it. What a rebuild should take from
this page is the *reasoning*: which quantities a shooter can quantize and which it cannot.

## State

Holds one player state, plus (in a variant that is compiled out, see the notes) a mask of
which fields changed since the last send.

Invariant: a freshly constructed holder is entirely zero except that the physics
orientation is set to an identity-like quaternion rather than all zeros, because an
all-zero quaternion is not a rotation and would produce a degenerate transform if sent
before the first real state arrives.

## The wire layout

**Contract** — writing and reading are exact mirrors. Everything is in one fixed order,
with no length prefix and no field tags, so the reader's order *is* the format.

```text
timestamp                     : 32 bits, raw
physics linear velocity x,y,z : 8 bits each, quantized over [-32, +32] metres/second
physics position x,y,z        : 32 bits each, full precision
model yaw                     : 32 bits, full precision
camera yaw, pitch, roll       : 8 bits each, quantized over [0, 2*pi]
logic acceleration            : a compressed unit direction with magnitude
--- then one small-field word, emitted as 1 to 4 bytes, least significant first ---
inventory active slot         : 4 bits, raw
body state flags              : 15 bits, raw
health                        : 8 bits, normalized to [0, 1]
radiation                     : 4 bits, normalized to [0, 1]
physics state enabled         : 1 bit
```

**Invariants**

- **Position is never quantized; velocity always is.** A quantized position drifts
  visibly and cannot be corrected; a quantized velocity is corrected by the next position
  update, and the velocity is only used to extrapolate between updates. This is the single
  most transferable decision on the page.
- The velocity range is clamped to ±32 metres per second *before* quantization, on the
  sending side, so the received value is always inside the range the decoder assumes. A
  player cannot exceed that speed by any normal means, and a physics anomaly that flings
  them faster is truncated rather than wrapped.
- The three camera angles are quantized over a full turn into 8 bits — about 1.4 degrees
  of resolution. That is coarse enough to see on a distant player's aim and is accepted
  because the aim that matters for hit registration is resolved on the shooter's own
  client, not from this field.
- The model yaw is *not* quantized, unlike the camera yaw. The body's facing drives the
  locomotion animation blend and quantization steps would make a turning player's legs
  judder.
- The small fields are accumulated into one 32-bit word and then emitted as **only as many
  bytes as the accumulated bit count needs** — one, two, three or four. The reader
  recomputes the same bit count from the same field list and consumes the same number of
  bytes. There is no length on the wire: the two sides agree because they run the same
  field list. Changing any field's width changes the framing.
- The field widths are the format: four bits of inventory slot caps the slots at sixteen,
  fifteen bits of body state must cover every movement flag, four bits of radiation gives
  sixteen visible levels.

## `pack` / `unpack` (normalized scalars)

**Contract** — compresses a value in the unit interval into a fixed number of bits and
back. Packing clamps to the unit interval, scales by the maximum representable value and
rounds to nearest. Unpacking divides by very slightly more than the maximum, so the result
is strictly inside the unit interval rather than reaching exactly one.

**Invariants** — packing has one special rule: **a non-zero input never packs to zero**,
provided more than one bit is available. Without it a player on a sliver of health would
be transmitted as dead, which is visible and wrong. The floor is asymmetric — the value
can still round *down* to the smallest representable non-zero step — and that asymmetry is
deliberate: the only value that must be exact is zero.

```text
FUNCTION pack(value, bits) -> int
  value = clamp(value, 0, 1)
  maximum = 2^bits - 1
  result = round(maximum * value)
  IF bits > 1 AND result == 0 AND value != 0 THEN result = 1   # never lose "barely alive"
  RETURN clamp(result, 0, maximum)

FUNCTION unpack(packed, bits) -> real
  RETURN packed / (2^bits - 1 + epsilon)    # epsilon keeps the result below 1
```

**Notes** — the epsilon means a fully packed value decodes to slightly less than one, so a
player at full health receives as very slightly wounded. Nothing in the game reads that
difference, but a rebuild reproducing the format should know the asymmetry is there.

## `relevant`

**Contract** — takes a fresh state, stores it, and answers whether it is worth sending. In
the shipped configuration the answer is always yes and every field is always written.

**Notes** — the file contains a complete, compiled-out delta scheme: a mask of which
fields differ from the last sent state, written as a leading 32-bit word, with each field
conditional on its bit. Every conditional in the reader and the writer is the remnant of
it, and in the shipped build every condition is constant-true. **The mask's sense is
inverted from what the writer expects** — it is set for fields that are *similar* to the
previous state, and the writer then sends exactly those — so the scheme as written sends
the unchanged fields and drops the changed ones. That is almost certainly why it is
disabled, and a rebuild wanting delta compression should write it fresh rather than
enabling this.

## Field-packing helpers

**Contract** — append a value of a given width to a word at a running bit offset, and read
one back the same way. Both assert the running offset never exceeds 32 bits, which is the
hard ceiling on the small-field word: adding a field that pushes the total past 32 bits
silently truncates, and the assertion is the only thing that catches it.
