# src/xrGame/filereceiver_node.cpp

> One inbound transfer: accumulate arriving chunks into a file or a memory buffer, learn the expected size from the first chunk, and know when it is finished.

**Needs** — [`filereceiver_node.h`](filereceiver_node.h.md) · [`filetransfer_common.h`](filetransfer_common.h.md) · [`xrCore/buffer_vector.h`](../xrCore/buffer_vector.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T1: it writes received bytes straight out of the packet buffer without copying, and the destination may be a file handle

## Purpose

The receiving half of one transfer session. It owns the destination — a file it opened, or a
memory buffer somebody else owns — appends each arriving chunk to it, and reports
completion. It knows nothing about peers, timeouts or the protocol's commands; the sites in
[`file_transfer.cpp`](file_transfer.cpp.md) own all of that.

The one piece of protocol it does own is the **header in the first chunk**, which carries
the total size and a caller-defined parameter. That has to live here because "is this the
first chunk" is a property of the destination's write position, which only this object
knows.

## State

```text
RECORD FileReceiverNode
  file_name            : text            # only meaningful for the file destination
  writer               : Writer          # the destination; owned only when it is a file
  is_writer_memory     : bool            # fixed at construction; decides who closes it
  data_size_to_receive : int             # 0 until the first chunk's header is read
  user_param           : int             # opaque; carried through from the sender
  last_read_time       : int (ms)        # global clock at the last chunk; drives the timeout
```

Invariants:

- **`writer` position is the authoritative progress counter.** There is no separate
  byte count; "how much have I received" is "where is the writer", and "is it complete" is
  "does that equal the expected size". That is what makes the object restart-proof: nothing
  can get out of step with itself.
- **Ownership follows `is_writer_memory`.** A file destination was opened here and is closed
  here; a memory destination belongs to the caller and is left alone. The flag is fixed at
  construction and never changes.
- **A session with a zero expected size and a zero write position is legitimately "complete"**
  by the equality test. The sites guard against that by never asking before the first chunk.

## Construction

**Contract** — two forms. The file form opens a writer for the named path through the
virtual filesystem; the open may fail, leaving no writer, which the caller must check. The
memory form adopts a caller-supplied memory writer and never releases it. Both start with a
zero expected size, so the first chunk's header is awaited.

**Notes** — a failed file open is reported by the writer being absent rather than by an
error, which is why every call site in the sites immediately asks for the writer and tears
the session down if it is missing. A rebuild should return a result and delete the two-step.

## `receive_packet`

**Contract** — appends one arriving chunk. On the first chunk (write position zero) it first
reads the two header fields. Yields true when the destination now holds exactly the expected
number of bytes. Writes directly from the packet's own buffer — no intermediate copy. Stamps
the read time.

```text
FUNCTION receive_packet(packet) -> complete
  IF writer.position == 0                        # this is the first chunk
    IF packet has fewer than 8 bytes remaining
      data_size_to_receive = writer.position     # i.e. 0 — see Notes
      RETURN false
    data_size_to_receive = packet.read_int32()
    user_param           = packet.read_int32()

  remaining = packet.length - packet.read_position
  writer.write(packet.buffer at read_position, remaining)
  last_read_time = global_clock
  RETURN writer.position == data_size_to_receive
```

**Invariants** — everything left in the packet after the header is payload. There is no
per-chunk length field; the chunk's length *is* the rest of the message, which is why the
transport must preserve message boundaries. A rebuild over a stream transport has to add
framing this design does not have.

**Notes** — the short-first-packet branch is a guard against a malformed or truncated first
message: rather than read past the end, it sets the expected size to the current write
position (zero) and declines to complete. The session then sits there with a zero expected
size until the timeout in the sites reaps it. It is defensive rather than correct — a
rebuild should treat a short first chunk as a protocol error and abort the session
immediately, because the current behaviour costs a full timeout to notice.

"First chunk" is detected by the write position being zero rather than by a sequence number.
That is sound because the transport is reliable and ordered, and it would silently corrupt a
transfer over anything less. The dependence is worth naming: **this protocol requires
reliable, ordered, message-preserving delivery** and has no sequencing, retransmission or
framing of its own.

The user parameter is opaque here and is carried from the sender to whoever reads it off the
node afterwards. It is how a transfer's *meaning* travels alongside its bytes — which file
this is, which request it answers — without the transfer layer knowing anything about it.

## `is_complete` / `signal_callback` / `get_downloaded_size`

**Contract** — `is_complete` is the same equality the receive returns, safe to ask at any
time (false when there is no writer at all). `signal_callback` reports a status together
with the current and expected sizes to the caller's progress callback. `get_downloaded_size`
is the writer's position.

**Notes** — the progress callback is invoked *by the sites*, not from inside the receive
path, which is what lets a callback tear down its own session safely: the site is between
operations when it calls.

## `split_received_to_buffers`

**Contract** — free function. Takes a fully received byte range that was sent as a *list of
buffers* and splits it back into that list, as views into the original bytes. Each buffer
was written with a four-byte length prefix by the sending side; this walks those prefixes.
Copies nothing — the views alias the caller's memory, which must outlive them.

```text
FUNCTION split_received_to_buffers(data, size) -> list of views
  reader = over (data, size)
  WHILE NOT reader.at_end
    length = reader.read_int32()
    views.append(view at reader.position, length)
    reader.seek(reader.position + length)
```

**Notes** — this is the exact inverse of the buffer-list sender in
[`filetransfer_node.cpp`](filetransfer_node.cpp.md), and the length prefix is the only thing
tying them together. It is the one place in the transfer layer where the *payload* has
structure; everywhere else the payload is opaque bytes. A rebuild must keep the prefix width
at four bytes, because both halves are on the wire.

No validation: a corrupt length walks the reader past the end. The transport is trusted to
have delivered what was sent.
