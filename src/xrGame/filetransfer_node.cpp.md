# src/xrGame/filetransfer_node.cpp

> One outbound transfer: four kinds of source behind one reading interface, plus the adaptive chunk size that decides how much goes out per update.

**Needs** — [`filetransfer_node.h`](filetransfer_node.h.md) · [`filetransfer_common.h`](filetransfer_common.h.md) · [`Level.h`](Level.h.md) · [`xrGame/xrServer.h`](xrServer.h.md) · [`xrCore/buffer_vector.h`](../xrCore/buffer_vector.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — reached through its declarations in [`filetransfer_node.h`](filetransfer_node.h.md); callers name that, not this file.
**Tier floor** — T1: sources are read as raw byte ranges straight into a packet, with a stack scratch buffer and a hard packet-size limit

## Purpose

The sending half of one transfer session. Two separable things live here.

The first is an **abstraction over where the bytes come from**. Four sources ship: a file on
disk, a fixed block of memory, a list of separately-sized buffers, and — the odd one — a
memory buffer that is *still being written to* while it is being sent. Each answers the same
five questions (give me a chunk, am I at the start, how big am I, how far have I got, am I
open), so the transfer node above them is source-agnostic.

The second is the **chunk-size controller**, the only flow control in the whole file
transfer system.

## State

```text
RECORD FileTransferNode
  reader                    : FileReader   # one of the four sources, owned
  chunk_size                : int (bytes)  # the current rate; bounded by the shared constants
  last_peak_throughput      : int (bytes/s) # the peak seen at the previous adjustment
  last_chunksize_update_time: int (ms)
  user_param                : int          # opaque; travels in the first chunk's header
  process_callback          : progress callback
```

Invariant: `chunk_size` stays within the shared minimum and maximum after every adjustment.
The clamp is applied at the end of the adjustment, not at each branch, so an intermediate
value may be out of range within the call.

## `calculate_chunk_size`

**Contract** — adjusts the chunk size from the connection's peak and current throughput.
Does nothing more than once per second. Two behaviours, depending on whether the peak
throughput is still rising.

```text
FUNCTION calculate_chunk_size(peak_throughput, current_throughput)
  IF global_clock - last_update < 1000 ms: RETURN

  IF last_peak_throughput < peak_throughput          # the link is still opening up
    chunk_size = chunk_size + data_min_chunk_size    # additive increase
  ELSE                                               # the peak has stopped rising
    IF authoritative side
      chunk_size = data_max_chunk_size               # see Notes
      RETURN                                         # note: without stamping the time
    IF global_clock - last_update < 3000 ms: RETURN
    chunk_size = random in [data_min_chunk_size, data_max_chunk_size]

  clamp chunk_size to [data_min_chunk_size, data_max_chunk_size]
  last_peak_throughput = peak_throughput
  last_update          = global_clock
```

**Notes** — this is the whole congestion control and it is unusual enough to be worth
stating plainly.

*While the observed peak keeps rising, the chunk size grows by one minimum chunk per
second.* Additive increase, 128 bytes at a time, capped at 4096 — so a transfer takes about
half a minute to reach full rate from the minimum. Deliberately slow: file transfer shares
the connection with the game, and the game must not stutter while a map downloads.

*Once the peak stops rising, the sender picks a new chunk size uniformly at random within
the bounds*, no more than once every three seconds. That is a genuinely strange policy — it
is neither multiplicative decrease nor a hold — and its effect is to keep probing the whole
range rather than converging. The plausible reading is that random re-probing avoids several
clients synchronising on the same rate against one server, but nothing in the source says
so. **This is the least explicable decision in the file transfer code**; a rebuild is better
served by a conventional decrease-on-congestion rule, and will not break the protocol by
changing it, because the chunk size is a purely local choice.

*The server never does any of that.* On the authoritative side the chunk size is pinned at
the maximum, because the server's uplink is assumed to be the good one and its transfers
(pushing a map to a joining client) should finish quickly. That branch also returns without
stamping the update time, so the one-second gate is re-evaluated against a stale timestamp
every call — harmless, since the branch is idempotent.

The peak throughput is the transport's own measurement, read from the per-client statistics
by the caller and passed in; this object never looks at the connection itself.

## `make_data_packet`

**Contract** — writes the next chunk into a packet. On the first chunk it prepends the total
size and the opaque caller parameter. Yields true when the source is exhausted. The packet
already carries the message header and command byte, written by the caller.

```text
FUNCTION make_data_packet(packet) -> is_last
  IF reader.is_first_packet
    packet.write_int32(reader.size)
    packet.write_int32(user_param)
  RETURN reader.make_data_packet(packet, chunk_size)
```

**Invariants** — the header is written exactly once per transfer, identified by the source's
own read position being zero. The receiver detects it the same way, from its write position
— the two ends agree without exchanging a sequence number, which is why the transport must
be reliable and ordered.

## `is_complete` / `is_ready_to_send` / `opened`

**Contract** — `is_complete` is "read position equals size", false when the source failed to
open. `is_ready_to_send` is its negation, with the source asserted open. `opened` asks the
source.

**Notes** — `is_ready_to_send` being exactly "not complete" means the node is always willing
to send; there is no pacing here. Pacing is entirely the chunk size, as above.

## `disk_file_reader` / `memory_reader`

**Contract** — the two simple sources. Each reads up to a chunk of bytes from its underlying
reader into the packet and reports whether it hit the end. The disk source opens through the
virtual filesystem at construction and closes at destruction; the memory source wraps a
caller-owned byte range.

```text
FUNCTION make_data_packet(packet, chunk_size) -> at_end
  to_write = min(chunk_size, reader.remaining)
  scratch  = stack buffer of to_write bytes
  REQUIRE to_write < packet_size_limit - packet.write_position
  reader.read(scratch, to_write)
  packet.write(scratch, to_write)
  RETURN reader.at_end
```

**Notes** — the copy through a stack scratch buffer is pure incidental: the reader's
interface takes a destination pointer and the packet's takes a source pointer, and neither
will hand over its own. A rebuild whose reader can write into a caller-provided span does
this with no copy at all. What *is* load-bearing is the size check: the packet has a hard
maximum, the chunk size is bounded well below it, and the check catches a caller that has
already filled the packet with something else.

The two sources are otherwise identical, and a rebuild with a uniform byte-source
abstraction needs one, not two.

## `buffers_vector_reader`

**Contract** — sends a list of separately-sized buffers as one stream, each prefixed with
its four-byte length, packing as many whole and partial buffers into each chunk as will fit.
Reports progress against a total that *includes* the length prefixes. Takes a copy of the
buffer list (the views, not the bytes) at construction.

```text
FUNCTION accumulate_size()
  total = sum over buffers of (buffer.length + 4)

FUNCTION make_data_packet(packet, chunk_size) -> at_end
  LOOP
    prefix = (we are at the start of the front buffer) ? 4 : 0
    rest   = (front.length - offset_into_front) + prefix
    IF chunk_size <= rest AND chunk_size > prefix
      write chunk_size bytes from the front buffer ; BREAK      # partial
    IF chunk_size <= prefix
      BREAK                                                     # see Notes
    write rest bytes from the front buffer                      # finishes it
    chunk_size = chunk_size - rest
  RETURN size == tell

FUNCTION write_from_front(packet, count)
  IF at the start of the front buffer
    packet.write_int32(front.length)      # the length prefix
    count = count - 4 ; completed = completed + 4
  packet.write(front.bytes at offset, count)
  offset = offset + count
  IF offset == front.length
    pop the front buffer ; completed = completed + front.length ; offset = 0
```

**Notes** — the length prefix is what
[`filereceiver_node.cpp`](filereceiver_node.cpp.md)'s splitter reads back, and including it
in the reported total is what makes the receiver's completion test line up: the receiver
counts raw bytes and knows nothing about the structure.

The "chunk too small for even a prefix" branch sends *nothing this update* and waits for a
larger chunk. It is reachable only when the chunk size is at or below four bytes, which the
minimum of 128 forbids, so it is dead in practice.

The loop's own termination condition is written as "while the remaining chunk is zero",
which is never true at the test point; every real exit is one of the two breaks. That is a
bug in shape rather than in effect — the last write always consumes the chunk exactly or
overshoots into a break — but a rebuild should write the loop as "while bytes remain in the
chunk and buffers remain".

## `memory_writer_reader`

**Contract** — the streaming source: reads from a memory buffer that is *still being
appended to* by somebody else, up to a maximum size fixed at construction. Sends whatever
has appeared since the last chunk; sends nothing when nothing new has appeared, and reports
completion only when the read position reaches the declared maximum.

```text
FUNCTION make_data_packet(packet, chunk_size) -> is_last
  available = writer.current_size - read_position
  IF available == 0
    RETURN read_position == declared_max      # see Notes
  to_write = min(chunk_size, available)
  REQUIRE to_write < packet_size_limit - packet.write_position
  packet.write(writer.bytes at read_position, to_write)
  read_position = read_position + to_write
  RETURN read_position == declared_max
```

**Notes** — this exists for relaying: a server forwarding a file it is itself receiving
starts sending before it has the whole thing. The declared maximum is therefore the
*promised* size, known in advance from the original transfer's header, while the writer's
current size is what has actually arrived.

The empty-available early return is documented in the source as a fix for exactly the case
that makes this source tricky. Readiness is judged by the node above as "read position is
less than size", and size here is the *promised* size — so a relay that has caught up with
its own inbound stream still looks ready, gets asked for a chunk, and has nothing. Returning
false there would keep the session alive with no progress; returning the completion test
lets it finish correctly if the promise has in fact been met. A rebuild should make
readiness ask the source rather than compare two numbers the source did not supply.

Note also that this source writes straight out of the other object's buffer with no copy and
no lock. It is safe only because both ends run on the same thread, interleaved between
updates.

## Construction and destruction

**Contract** — four constructors, one per source kind, each taking the initial chunk size and
the progress callback; the three memory-backed ones also take the opaque caller parameter.
Destruction releases the source.

**Notes** — the *initial* chunk size differs by caller, not by source: the sites start server
transfers at the maximum and client transfers at the minimum
([`file_transfer.cpp`](file_transfer.cpp.md)). That, together with the server's pinned
maximum in the rate controller, is the asymmetry of the whole design — the server pushes
hard, the client trickles.
