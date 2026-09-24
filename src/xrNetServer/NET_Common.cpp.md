# src/xrNetServer/NET_Common.cpp

> Turns a stream of small messages into a stream of compressed datagrams, and back — the
> envelope that both ends of the connection share.

**Needs** — [`NET_Common.h`](NET_Common.h.md) · [`NET_Messages.h`](NET_Messages.h.md) · [`NET_Compressor.h`](NET_Compressor.h.md) · [`xrCore/net_utils.h`](../xrCore/net_utils.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport) · [Seam: Threads](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — reached through its declarations in [`NET_Common.h`](NET_Common.h.md); callers name that, not this file.
**Tier floor** — T1: it composes a byte layout in place and is on the per-frame send path for
every connected client, so it must not allocate.

## Purpose

World updates are small and frequent; datagrams have a fixed cost. This file exists so nobody
above it has to think about that: callers send messages one at a time and the batching,
compression and splitting happen underneath. It is the only place in the engine that knows
several messages can share a datagram.

Sender and receiver live in one file because they are one format read in two directions. A
rebuild that separates them must keep the envelope layout in a single description.

## State

```text
RECORD EnvelopeHeader          # three bytes, no padding, at the front of every datagram
  tag            : int (8-bit)    # merged (0xE1) or single (0xE0)
  unpacked_size  : int (16-bit)   # payload size BEFORE compression

RECORD AccumulationBuffer
  buffer      : MessageBuffer    # capacity 16384 bytes, the datagram limit
  last_flags  : ChannelFlags     # the channel every message currently inside was queued with

RECORD SenderState
  normal   : AccumulationBuffer   # everything, by default
  reliable : AccumulationBuffer   # reliable traffic only, in the separating policy
  lock     : Mutex                # both buffers, one lock

CONSTANT max_envelope = 32768     # scratch size for a decompressed payload;
                                  # twice the datagram limit, so expansion cannot overrun
```

**Invariants** — every message inside one accumulation buffer was queued with the same channel
flags, ignoring the `immediate` bit. This is what the flush-on-flags-change rule exists to
preserve, and it is the reason a rebuild may not simply batch by size: an envelope is
transmitted once, with one set of guarantees, and every message inside it inherits them.

The compressor is a single process-wide instance shared by both buffers and by the receiver,
serialized by its own lock. That is a memory economy, not a design decision; a rebuild may
give each endpoint its own.

## `MultipacketSender`

**Contract** — an endpoint mixes this in and supplies one operation: *transmit these bytes on
this channel*. In return it gets two: queue a message, and flush. Queuing may transmit (if the
buffer is full, the channel changed, or the caller asked for immediacy) and is otherwise
cheap. Both operations take the sender's lock, because the transport's callback thread and the
simulation thread both reach them. Neither allocates. Flushing an empty buffer does nothing.

```text
FUNCTION send_message(message : bytes, flags : ChannelFlags, timeout)
  LOCK sender DURING
    buf = normal
    CASE guaranteed_packet_mode OF
      ignore    : clear the reliable bit in flags          # force everything unreliable
      separate  : IF flags.reliable THEN buf = reliable    # keep the two streams apart
      default   : # one buffer for everything
    END

    # An envelope carries one channel. Anything that would break that, or overflow
    # the datagram, closes the current envelope first.
    IF buf.used + size(message) + 2 >= datagram_limit
       OR flags differs from buf.last_flags (ignoring the immediate bit)
       OR flags.immediate
    THEN flush(buf, timeout)

    append 16-bit size(message) to buf
    append message bytes to buf

    IF flags.immediate THEN flush(buf, timeout)
    buf.last_flags = flags
```

**Notes** — the `+ 2` in the overflow test accounts for the length prefix the message is about
to acquire, and the test is `>=` rather than `>`, so the buffer is flushed one byte before it
strictly needs to be. Harmless.

Note the ordering trap in the immediacy case: the message is appended and *then* flushed, so
an immediate message still travels with whatever was already buffered. That is correct — the
earlier messages share its channel, by the flags-change rule — but it means "immediate" means
*"send now"*, not *"send alone"*.

`last_flags` is updated after the flush, not before, which is what makes the comparison
meaningful for the *next* message.

## `flush`

**Contract** — called with the sender's lock held. Compresses the accumulated payload, writes
the header in front of it, hands the whole thing to the endpoint's transmit operation, and
empties the buffer. Never partially transmits: a payload either goes whole or the buffer keeps
it.

```text
FUNCTION flush(buf, timeout)
  IF buf is empty THEN RETURN

  worst = compressor.worst_case_size(buf.used)
  REQUIRE worst fits in the scratch envelope and in a 16-bit size field

  size = compressor.compress(scratch + header_size, buf.contents)
  scratch.tag           = envelope_merged
  scratch.unpacked_size = buf.used         # size BEFORE compression - the receiver
                                           # needs it to know when it has split enough
  transmit(scratch, header_size + size, buf.last_flags, timeout)
  buf.used = 0
```

**Notes** — the size written into the header is the *uncompressed* size, and it is the only
thing that tells the receiver how far to walk the split loop; the compressed length is implied
by the datagram's own length. A rebuild must keep that asymmetry or add a second field.

The scratch buffer is a stack allocation of 32 kilobytes on the sending thread, taken on every
flush. That is an artifact of avoiding an allocator on the send path; the decision that
survives is *no allocation while flushing*, not the stack array.

Both accumulation buffers are flushed when the caller asks for a flush explicitly — the
reliable one even in the policies that never fill it, which costs an empty check and keeps the
call site from having to know which policy is active.

A diagnostic mode, enabled by a command-line switch, appends every pre-compression payload to
a capture file with a four-byte signature and a 16-bit length per record. It is a capture
format, not a protocol, and a rebuild is free to choose its own.

## `MultipacketReciever`

**Contract** — an endpoint mixes this in and supplies one operation: *here is one message*. In
return it gets one: here is a datagram, split it. Decompresses into a scratch buffer, then
walks the payload handing out messages one at a time. An unrecognized tag causes the whole
datagram to be dropped silently. Runs on whatever thread the transport delivers on.

```text
FUNCTION receive_datagram(datagram : bytes, from)
  header = first three bytes
  IF header.tag is neither merged nor single THEN RETURN      # silently
  payload = compressor.decompress(datagram after header)

  merged = (header.tag = envelope_merged)
  consumed = 0
  WHILE consumed < header.unpacked_size
    IF merged THEN
      size = take 16-bit length from payload
      consumed = consumed + 2
    ELSE
      size = header.unpacked_size
    deliver(payload slice of `size` bytes, from)
    consumed = consumed + size
```

**Invariants** — the loop terminates only because `unpacked_size` is trustworthy. It comes
from the datagram, so it is attacker-controlled; the checksum inside the compressed payload
is what makes it trustworthy in practice, and the receiving endpoints additionally refuse
datagrams larger than the buffer limit before reaching here. A rebuild should validate the
walk against the *actual* decompressed length as well, which this code does not.

**Notes** — the single-message tag is handled but never produced. It is one branch of dead
compatibility, and a rebuild that only ever merges can drop the tag byte's second value.

The dropped-on-bad-tag case is silent, with no counter and no log. On a link that is
delivering garbage this looks identical to a link delivering nothing.
