# src/xrNetServer/NET_Common.h

> Declares the envelope sender/receiver pair that every endpoint mixes in, and holds the
> module's compile-time switches and wire tag values.

**Needs** — [`xrCore/net_utils.h`](../xrCore/net_utils.h.md) · [`NET_Shared.h`](NET_Shared.h.md)
**Used by** — [`NET_Client.cpp`](NET_Client.cpp.md) · [`NET_Client.h`](NET_Client.h.md) · [`NET_Common.cpp`](NET_Common.cpp.md) · [`NET_Compressor.cpp`](NET_Compressor.cpp.md) · [`NET_PlayersMonitor.h`](NET_PlayersMonitor.h.md) · [`NET_Server.cpp`](NET_Server.cpp.md) · [`NET_Server.h`](NET_Server.h.md) · [`NET_Client.cpp`](empty/NET_Client.cpp.md) · [`NET_Client.h`](empty/NET_Client.h.md) · [`NET_Server.cpp`](empty/NET_Server.cpp.md) · [`NET_Server.h`](empty/NET_Server.h.md)
**Tier floor** — T1: it fixes the tag byte values that appear on the wire.

## Purpose

Declares the surface implemented in [`NET_Common.cpp`](NET_Common.cpp.md), and — because
there is nowhere better — carries the module's configuration switches and the constants that
appear in the envelope header.

## Exported units

- **`MultipacketSender`** — the accumulate-and-flush half of the envelope. An endpoint mixes
  it in and supplies one operation: hand these bytes to the transport. Contract in
  [`NET_Common.cpp`](NET_Common.cpp.md).
- **`MultipacketReciever`** — the split half. An endpoint mixes it in and supplies one
  operation: here is one message, interpret it. Contract in
  [`NET_Common.cpp`](NET_Common.cpp.md).
- **`GameDescriptionData`** — the blob a server attaches to its session so a prospective
  client can read it *before* connecting: map name, map version, and a URL to download the map
  from. Three fixed-size text fields, transmitted as a memory image. Its size is part of the
  session advertisement, so both ends must agree on it exactly.
- **`psNET_GuaranteedPacketMode`** — selects which of the three accumulation policies the
  sender uses (share one buffer; strip reliability from everything; keep reliable and
  unreliable traffic in separate buffers).

## The wire tags

```text
CONSTANT envelope_merged        = 0xE1   # payload is a sequence of length-prefixed messages
CONSTANT envelope_single        = 0xE0   # payload is exactly one message
CONSTANT payload_compressed     = 0xC1   # payload bytes are compressed
CONSTANT payload_uncompressed   = 0xC0   # payload bytes are verbatim
```

Paired values differing in one bit, so a corrupted tag is more likely to land on an invalid
value than on the other member of its pair. Envelopes whose tag is neither of the first two
are discarded without comment.

## Notes

The switches in this file — whether to merge, whether to compress, which compressor, whether
to checksum, whether to trace — are compile-time constants, every one of them fixed to the
same value in every configuration that ships. They are not configuration; they are the
fossilized record of decisions already made. The three that are still *live* decisions, and so
survive into a rebuild, are: envelopes are merged, payloads are compressed opportunistically,
and compressed payloads carry a checksum.

One switch selects between two compressors — a fast byte-oriented one and a much slower
statistical one. Only the fast one is reachable in any shipping build. The slower path exists
in the source and a rebuild should not carry it.
