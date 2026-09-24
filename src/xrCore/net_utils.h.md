# src/xrCore/net_utils.h

> The network packet: a fixed-capacity byte buffer with a typed write and read cursor, and the optional mirror that dumps every field to a text stream for debugging.

**Needs** — [`NET_utils.cpp`](NET_utils.cpp.md) · [`xr_types.h`](xr_types.h.md) · [`client_id.h`](client_id.h.md) · [`xrstring.h`](xrstring.h.md) · [`../xrCommon/xr_string.h`](../xrCommon/xr_string.h.md) · [Seam: Networking transport](../../SYSTEM-REQUIREMENTS.md#seam-networking-transport)
**Used by** — [`xrCore.h`](xrCore.h.md) · [`xr_object_list.cpp`](../xrEngine/xr_object_list.cpp.md) · [`Message_Filter.cpp`](../xrGame/Message_Filter.cpp.md) · [`Message_Filter.h`](../xrGame/Message_Filter.h.md) · [`NET_Queue.h`](../xrGame/NET_Queue.h.md) · [`NET_Client.cpp`](../xrNetServer/NET_Client.cpp.md) · [`NET_Client.h`](../xrNetServer/NET_Client.h.md) · [`NET_Common.cpp`](../xrNetServer/NET_Common.cpp.md) · [`NET_Common.h`](../xrNetServer/NET_Common.h.md) · [`NET_Log.cpp`](../xrNetServer/NET_Log.cpp.md) · [`NET_Server.cpp`](../xrNetServer/NET_Server.cpp.md)
**Tier floor** — T1: the buffer is a byte-exact wire image written by unaligned stores of native-endian values, and both the network protocol and the save-game format are read back out of it.

## Purpose

Declares the packet type whose conversions live in [`NET_utils.cpp`](NET_utils.cpp.md). The header is substantive in one respect: it fixes the *shape* of the buffer and the rule that every write is a raw copy of the value's native byte image. That rule is why the format is little-endian-only and why every entity's serialized state — not just network traffic, but save games too, since both go through this type — is a memory image.

## State

```text
RECORD Packet
  data          : bytes           # exactly 16384 bytes, inline, never heap
  count         : int (32-bit)    # bytes written; the write cursor
  read_position : int (32-bit)    # the read cursor, independent of count
  received_at   : int (32-bit)    # arrival time, stamped by the transport
  write_allowed : bool            # see below
  mirror        : optional<FieldStream>   # debug echo of every typed field
```

**Invariants**

- **The buffer is 16384 bytes and is not dynamic.** Every write asserts that it fits; overflowing is a failure, not a resize. This is the packet size limit the transport and the entity serializers are written against.
- The buffer is byte-packed with no padding between fields; a written value occupies exactly its own width.
- Read and write cursors are independent. A packet is written then rewound then read; it is never both at once.
- **The mirror is all-or-nothing per typed field.** The `write_allowed` flag is set for the duration of a typed write and cleared after, so that a raw write reaching the buffer outside a typed call is detected when a mirror is attached — the mirror records *fields*, and a raw write would desynchronize it from the byte stream.

## The field mirror interface

**Contract** — An optional sink that receives the same sequence of typed values the packet does, in the same order, so the packet can be replayed as text. Implementors must support: rewind, one write per value type (real, 3- and 4-vectors, each integer width, null-terminated string), the matching reads, a sized string read, and a string skip.

**Invariants** — The mirror sees *values*, not bytes. Two packets that hold the same bytes for different reasons produce different mirror output, which is the point: it is a debugging aid for the save-game and network serializers, where a field-order mismatch between writer and reader is the usual bug and is invisible in a hex dump. Several operations deliberately have no mirror equivalent and assert if one is attached — the nested-chunk framing below is the main case, because its length prefix is patched after the fact and has no value-level meaning.

## Exported units

- **Construct from bytes** — copy an incoming datagram into the buffer and set the length.
- **Write start / write begin** — reset the cursor; the second also writes a 16-bit message type as the first field, which is the framing every message shares.
- **Raw write, positioned write, write tell** — the primitive every typed write goes through, plus the back-patching write used by chunk framing.
- **Typed writes** — real, 3- and 4-vector, each signed and unsigned integer width, each a raw copy of the native byte image.
- **Quantized writes** — a real into 8 or 16 bits over a caller-supplied range; an angle into 8 or 16 bits over a full turn; a unit direction into 16 bits; a scaled direction as a 16-bit unit direction plus a full real magnitude. Contracts in [`NET_utils.cpp`](NET_utils.cpp.md); these are the frozen-against-itself part of the wire format.
- **String write** — the bytes plus a terminator. A null string writes a single zero byte.
- **Matrix write** — four 3-vectors in the order first axis, second axis, third axis, translation. The fourth column is not transmitted.
- **Client identifier write/read** — a 32-bit value.
- **Nested chunk framing** — open reserves a length prefix of 8 or 16 bits and remembers its position; close computes the payload length, asserts it fits the prefix width, and patches it. This is how variable-length sub-records nest inside one packet.
- **Read start, read begin, seek, tell, advance, at end, elapsed** — the read cursor.
- **Typed reads** — the mirror of every write, in both an out-parameter form and a return-value form.
- **String reads** — into a raw buffer, into an owning string, into an interned string, into a sized buffer with truncation, and a skip.
