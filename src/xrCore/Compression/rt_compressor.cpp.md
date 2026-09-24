# src/xrCore/Compression/rt_compressor.cpp

> The fast compressor: what every archived file is inflated with, with per-thread scratch so level loading can parallelize.

**Needs** — [`rt_compressor.h`](rt_compressor.h.md) · [Seam: Compression](../../../SYSTEM-REQUIREMENTS.md#seam-compression) · [Seam: Threads, atomics and process services](../../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`rt_compressor.h`](rt_compressor.h.md)
**Tier floor** — T1: a fixed-size scratch buffer with thread-local storage duration and a specific alignment requirement.

## Purpose

The decompressor on the level-loading critical path. [`LocatorAPI.cpp`](../LocatorAPI.cpp.md) calls it for every compressed archive entry, so it runs thousands of times during a load and its throughput is the load time.

The one design decision here that matters is the **per-thread scratch buffer**: the compression library needs a work area, and giving each thread its own — rather than one shared behind a lock — is what lets several worker threads inflate archive entries concurrently. In a rebuild, this is the constraint to carry: the compressor must be callable from any thread with no coordination.

## State

```text
# Per thread, for the life of the thread:
scratch : bytes[library-published size for the fast level]
# invariant: must satisfy the library's alignment requirement, which is
#   stricter than a byte array's. It is declared as an array of an aligned
#   element type rounded up to cover the required size.
```

**Notes** — the scratch buffer is needed only by the *compressor*; the decompressor is handed it anyway and ignores it. Since the fast compressor is barely used (the archives are built with the strong one), the per-thread cost is almost pure waste — a rebuild should allocate it lazily or only for the compressing path.

## `rtc_initialize`

**Contract** — one-time library setup. Must succeed; failure is fatal.

## `rtc_compress` / `rtc_decompress`

**Contract** — one-shot in-memory compression and decompression. The destination capacity is passed in and the actual length is returned. The compressor uses the library's speed-optimized level. The decompressor uses the **unchecked** variant, which assumes the input is well-formed.

**Invariants** — the caller must know the exact decompressed length before calling, because the unchecked decompressor writes until the stream says stop and the capacity is only advisory. The archive directory stores that length ([`LocatorAPI.cpp`](../LocatorAPI.cpp.md)), which is what makes this safe there.

**Notes** — choosing the unchecked decompressor is a deliberate trade: it is measurably faster and every caller is reading bytes the engine itself produced. A rebuild decompressing anything that came over a network must use the bounds-checked form instead — see [`lzo_compressor.cpp`](lzo_compressor.cpp.md), which does exactly that for the same reason.
