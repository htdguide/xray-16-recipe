# src/xrGame/filetransfer_common.h

> The vocabulary of the in-band file transfer: the three commands on the wire, the two progress vocabularies, and the chunk-size bounds that decide how fast a transfer may go.

**Needs** — [`xrNetServer/NET_Messages.h`](../xrNetServer/NET_Messages.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`file_transfer.cpp`](file_transfer.cpp.md) · [`filereceiver_node.cpp`](filereceiver_node.cpp.md) · [`filereceiver_node.h`](filereceiver_node.h.md) · [`filetransfer_node.cpp`](filetransfer_node.cpp.md) · [`filetransfer_node.h`](filetransfer_node.h.md)
**Tier floor** — T3: constants, enumerations and two packet constructors

## Purpose

Multiplayer needs to move whole files between peers over the same connection the game runs
on — a map a client does not have, a screenshot, a demo. This file declares the small shared
vocabulary that both ends of that use, so the sender and the receiver cannot disagree.

Everything here is frozen by the wire: the command byte's three values and the message
identifier are part of the protocol.

## State

```text
ENUM Command                # the first byte of every file-transfer message
  receive_data     = 0      # this message carries a chunk
  abort_receive    = 1      # sender gives up; sent by the SOURCE
  receive_rejected = 2      # I do not want this; sent by the DESTINATION

ENUM SendingStatus          # reported to the sender's progress callback
  sending_data, sending_aborted_by_user, sending_rejected_by_peer, sending_complete

ENUM ReceivingStatus        # reported to the receiver's progress callback
  receiving_data, receiving_aborted_by_peer, receiving_aborted_by_user,
  receiving_timeout, receiving_complete
```

Two constants bound the adaptive chunk size:

```text
data_max_chunk_size = 4096 bytes    # about 80 KB/s at the transfer update rate
data_min_chunk_size = 128  bytes    # about 2.5 KB/s
```

**Notes** — the byte figures come with throughput figures in the source, and that pairing is
the load-bearing part: the chunk size is the *only* rate control in this system. There is no
timer and no token bucket — one chunk goes out per transfer update, so the chunk size
multiplied by the update rate **is** the transfer rate. The two constants therefore define
the entire range from "barely noticeable alongside gameplay traffic" to "as fast as the
protocol will carry". A rebuild that sends on its own schedule must reintroduce a rate limit
some other way, or file transfer will starve the game traffic sharing the connection.

The two status vocabularies are deliberately asymmetric. A sender can be rejected by its
peer but never times out; a receiver can time out but is never "rejected". That mirrors who
holds the initiative: the sender pushes, so only the receiver can refuse, and only the
receiver can be left waiting.

There is no "aborted by user" path on the sending side that anything actually signals; the
value exists in the enumeration and is never produced.

## `make_reject_packet`

**Contract** — composes a rejection message naming a peer. Used by a receiver that is
offered data it has no session for, and by a receiver tearing down an incomplete session.

```text
FUNCTION make_reject_packet(packet, peer)
  packet.begin(FILE_TRANSFER)
  packet.write_byte(receive_rejected)
  packet.write_int32(peer.value)
```

## `make_abort_packet`

**Contract** — composes an abort message naming a peer. Used by a sender tearing down an
incomplete transfer.

**Notes** — both messages carry a peer identifier in the same position as a data message's
source identifier, so the three commands share one frame shape. The identifier is sometimes
written as zero by callers that have no meaningful peer to name (a client talking to the
server, where the server is implied); the reader on that side ignores it. A rebuild should
not read a zero here as a valid peer.

## Progress callbacks

**Contract** — both sides report progress through a callback taking a status, the number of
bytes moved so far, and the total expected. Together those three are enough to drive a
progress bar and to distinguish every terminal outcome.

**Notes** — the *total* is not known to the receiver until the first chunk arrives (it is in
the first chunk's header), so a receiver's callback reports a total of zero until then. A
progress display must tolerate that.

## Buffer aliases

**Contract** — a mutable and a const view over a length-tagged byte range, used where a
transfer's source is a list of in-memory buffers rather than a file.
