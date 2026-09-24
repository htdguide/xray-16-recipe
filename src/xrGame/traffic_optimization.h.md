# src/xrGame/traffic_optimization.h

> Declares the two pre-trained model loaders and the flag set that says which compressor a
> session uses.

**Needs** — [`traffic_optimization.cpp`](traffic_optimization.cpp.md) · [`xrCore/Compression/compression_ppmd_stream.h`](../xrCore/Compression/compression_ppmd_stream.h.md) · [`xrCore/Compression/lzo_compressor.h`](../xrCore/Compression/lzo_compressor.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — [`Level.h`](Level.h.md) · [`traffic_optimization.cpp`](traffic_optimization.cpp.md) · [`xrServer_updates_compressor.h`](xrServer_updates_compressor.h.md)
**Tier floor** — T1: it names raw byte buffers and their alignment obligations.

## Purpose

Declares the surface implemented in
[`traffic_optimization.cpp`](traffic_optimization.cpp.md), plus the small flag set that is
the actual protocol-visible part of this module.

## State

```text
ENUM TrafficOptimization                 # a flag set, not an enumeration of alternatives
  none              = 0
  ppmd_compression  = bit 0
  lzo_compression   = bit 1
  last_change       = bit 2
```

**Invariants** — the values are **bit positions in a field that travels on the wire**, so
they are frozen by the network protocol: a client and a server must agree on which bit means
which compressor, and a session negotiates its optimizations by exchanging this field.

The two compression bits are independent rather than exclusive, so a session may in principle
have both set. `last_change` is a different kind of thing carried in the same field — a delta
optimization, "this update is unchanged since the last one" — which is why the type is a flag
set and not a choice of compressor.

```text
RECORD LzoDictionary
  data : bytes
  size : int
```

## Exported units

- `init_ppmd_trained_stream` / `deinit_ppmd_trained_stream` — load and release the trained
  statistical model. Release must recover the backing buffer from the stream before
  destroying it.
- `init_lzo` / `deinit_lzo` — load the dictionary and allocate sixteen-byte-aligned scratch
  memory, returning both the aligned pointer the compressor uses and the raw allocation that
  must be freed.
