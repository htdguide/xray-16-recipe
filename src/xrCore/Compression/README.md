# src/xrCore/Compression — the four compressors and what each is for

Part of chapter 6, [`src/xrCore`](../README.md). Fourteen files covering three distinct
compression schemes plus the memory pool one of them needs, and one decision per scheme
about where it may be used.

## What this module is responsible for

The engine compresses in four places and they have nothing in common but the word.

- **Archive payloads** are inflated at load time, thousands of times per level, on several
  threads. Speed is everything and ratio barely matters, because the archive was built once
  by a tool. This is the fast dictionary compressor.
- **Network payloads** are compressed per packet, where the ratio pays directly for
  bandwidth and the work is tiny. This is the same library's strong mode, with a *shared
  dictionary* both ends hold so that a small packet still compresses.
- **Save games** are one large, highly structured snapshot compressed once and decompressed
  once. Ratio is worth almost any amount of time, because the cost shows up as load time and
  disk space. This is the statistical coder — six of the fourteen files.
- **Archive directories and legacy chunk bodies** use the engine's own older codec, which
  lives outside this directory in [`../LzHuf.cpp`](../LzHuf.cpp.md) and is named here only
  so a reader does not go looking for it.

The module owns the choice of scheme per use, the parameters each is run with — which for
the statistical coder are *part of the format* — and the serialization that keeps the
adopted code from touching files directly.

## Where it sits

It rests on [Seam: Compression](../../../SYSTEM-REQUIREMENTS.md#seam-compression) for the
fast dictionary compressor and the standard deflate, on
[`../Threading/Lock.hpp`](../Threading/Lock.hpp.md) for the serialization the statistical
coder forces, and on the allocator. It is consumed by the virtual filesystem
([`../LocatorAPI.cpp`](../LocatorAPI.cpp.md)), the network layer (chapter 9), and the save
system (chapter 23).

Read [`../LzHuf.cpp`](../LzHuf.cpp.md) alongside this directory: the four generations of
archive format use different combinations of these codecs and the filesystem twin says
which.

## The load-bearing ideas

**Ratio and speed are not a slider here; they are four separate decisions.** The single
most common mistake a rebuild can make in this directory is to unify them. Swapping the
save compressor into the archive path multiplies level load time; swapping the archive
compressor into the save path multiplies save size. Each twin states which constraint it
serves.

**The fast compressor's format is frozen and the strong one's is not.** Shipped archives
are already compressed and their bytes cannot be renegotiated. Network payloads are
compressed by this codebase on both ends, so a rebuild may use anything — provided it
version-gates the protocol.

**The statistical coder is three cooperating files and one global.** [`Model.cpp`](Model.cpp.md)
decides probabilities, [`Coder.hpp`](Coder.hpp.md) turns them into bits,
[`SubAlloc.hpp`](SubAlloc.hpp.md) holds the tree — and all three keep their state in
file-scope variables, which is why every compression in the process is serialized behind one
lock. That is a property of the adopted implementation, not of the algorithm. A rebuild that
makes the coder an object deletes the lock and gains concurrent saves.

**For the statistical coder, the parameters are the format.** The model order, the
restoration policy, and even the size of the memory pool all change *when* the model resets,
and both sides must reset at the same byte. They are named once, in
[`ppmd_compressor.cpp`](ppmd_compressor.cpp.md), and must not be treated as tuning.

**Per-thread scratch is what makes parallel level loading possible.** The fast compressor
needs working memory; giving each worker its own is the difference between loading a level
on eight threads and on one. See [`rt_compressor.cpp`](rt_compressor.cpp.md).

## The twins

| File | Role |
|---|---|
| [`rt_compressor.h`](rt_compressor.h.md) | Declares the two real-time compressors: the fast one used everywhere, the strong one used on the wire. |
| [`rt_compressor.cpp`](rt_compressor.cpp.md) | **The fast compressor** — what every archived file is inflated with, with per-thread scratch so level loading parallelizes. |
| [`rt_compressor9.cpp`](rt_compressor9.cpp.md) | **The strong compressor**, with the shared dictionary the multiplayer protocol compresses against. |
| [`lzo_compressor.h`](lzo_compressor.h.md) | Declares the dictionary-assisted compressor used for network payloads. |
| [`lzo_compressor.cpp`](lzo_compressor.cpp.md) | Thin pass-through to the library's dictionary-assisted entry points. |
| [`ppmd_compressor.h`](ppmd_compressor.h.md) | Declares the statistical compressor's six entry points: plain, model-trained, chunked-with-yield. |
| [`ppmd_compressor.cpp`](ppmd_compressor.cpp.md) | **The three format parameters**, the process-wide lock, and the chunked form that keeps a long save from freezing a frame. |
| [`PPMd.h`](PPMd.h.md) | The adopted coder's own interface: pool bring-up, encode, decode, and the restoration policy. |
| [`PPMdType.h`](PPMdType.h.md) | Its compile-time configuration: signature, order ceiling, and the stream binding. |
| [`Model.cpp`](Model.cpp.md) | **The statistical model**: a suffix tree of byte contexts with information inheritance, secondary escape estimation, and a memory-exhaustion policy. The longest page in the directory, and frozen. |
| [`Coder.hpp`](Coder.hpp.md) | **The carryless range coder**: how a probability becomes bits. Frozen, including the halved carry floor. |
| [`SubAlloc.hpp`](SubAlloc.hpp.md) | **The coder's private pool**: one block carved from both ends, 38 size classes, and the exhaustion that triggers restoration. |
| [`compression_ppmd_stream.h`](compression_ppmd_stream.h.md) | The byte cursor that makes a memory buffer look like the file handle the coder expects. |
| [`compression_ppmd_stream_inline.h`](compression_ppmd_stream_inline.h.md) | Its operations, and the read/write asymmetry the coder depends on. |
