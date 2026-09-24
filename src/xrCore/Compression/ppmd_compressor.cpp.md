# src/xrCore/Compression/ppmd_compressor.cpp

> Wraps the statistical coder: picks its three tuning parameters, serializes access to its global state, and splits long work into yieldable pieces.

**Needs** — [`ppmd_compressor.h`](ppmd_compressor.h.md) · [`PPMd.h`](PPMd.h.md) · [`compression_ppmd_stream.h`](compression_ppmd_stream.h.md) · [`../Threading/Lock.hpp`](../Threading/Lock.hpp.md)
**Used by** — [`ppmd_compressor.h`](ppmd_compressor.h.md)
**Tier floor** — T1: the coder underneath owns one large fixed pool and a graph of raw pointers into it; the pool size is a hard budget chosen here.

## Purpose

The statistical compressor is the highest-ratio, slowest option the engine has, used where size matters far more than speed — principally the save-game snapshot. This file is everything around it: the three tuning constants, the serialization that its global state forces, and the chunked forms that keep a long compression from freezing the frame.

## The three constants

```text
pool_size       = 32        # megabytes, for the coder's private memory pool
model_order     = 8         # how many preceding bytes of context the model
                            # conditions on
restoration     = restart   # what to do when the pool fills
```

**Invariants** — these three are **part of the format**. The decoder must use the same order and the same restoration policy as the encoder, because both sides build the same model from the same byte history and any difference desynchronizes them immediately. The pool size must be at least as large on the decoding side, for the same reason: running out at a different point changes when the model restarts.

**Notes** — order 8 is a deliberate midpoint. Higher orders compress structured data better and cost memory and time superlinearly; the coder allows up to 16. Thirty-two megabytes is a large permanent allocation for a 2007 engine, and it is taken because the save snapshot is megabytes of highly structured data where the ratio pays for itself in load time.

*Restart* — throw the model away and begin again — is chosen over the two alternatives (prune the tree, which the source notes is nearly twice as slow; or freeze it, which the source calls dangerous). Restart is the only one of the three that is obviously deterministic on both sides.

## `ppmd_initialize`

**Contract** — on first call, create the serialization lock and allocate the coder's pool; abort the process if the pool cannot be allocated. On every call, if a trained model is installed, rewind its cursor so the next operation reads it from the start.

**Notes** — aborting on allocation failure rather than reporting it is a real decision: the alternative is a save that silently does not happen. It is also the only path in the engine that exits without going through the crash handler.

The lock is created on first use and never destroyed, which the source itself flags as a hazard at process teardown. In a rebuild the lock is a static with no destruction concern.

## `ppmd_compress` / `ppmd_decompress`

**Contract** — compress or decompress a whole buffer into a whole buffer, under the global lock, with an empty starting model. Compression reports the output length **plus one**; decompression reports the output length exactly.

**Invariants** — the off-by-one on the compress side is not a bug to fix: the range coder's flush writes four bytes, the last of which the cursor's offset does not account for in the same way the decoder's does. Reporting one byte more than the cursor says is what makes the reported length cover everything the decoder will read. Changing it truncates the last byte and corrupts the tail of every compressed buffer.

**Notes** — the whole operation holds one process-wide lock, because the coder keeps its model, its pool and its range-coder registers in file-scope variables. That is a property of the adopted algorithm's implementation, not of the algorithm. **A rebuild should make the coder's state an explicit object**, at which point the lock disappears and two saves can compress concurrently.

## `ppmd_trained_compress` / `ppmd_trained_decompress`

**Contract** — install a caller-supplied serialized model as the starting state, run the operation, and restore whatever model was installed before. The installed model is read from the start each time, which is what the rewind in initialization is for.

**Invariants** — the previous model is saved and restored around the call, so these nest. They still hold the global lock for the duration, so the nesting is only ever within one thread.

## `ppmd_compress_mt` / `ppmd_decompress_mt`

**Contract** — do the same work in pieces, invoking the caller's callback between them, so the calling thread can yield, pump a message queue or update a progress bar. Compression splits the *input* into fixed 100 KiB pieces, each compressed as an independent stream appended to the output. Decompression walks the input, decoding one stream at a time and advancing by however much that stream consumed.

```text
FUNCTION compress_in_pieces(dest, src, callback) -> int
  LOCK ppmd_lock DURING
    produced := 0
    WHILE src IS NOT EXHAUSTED
      piece := next min(100 KiB, remaining) bytes OF src
      n := encode(piece) INTO dest AT produced      # an INDEPENDENT stream
      produced := produced + n
      IF callback EXISTS THEN callback()
    RETURN produced
```

**Invariants** — each piece is an independent stream, so the model is **reset at every piece boundary**. That costs ratio — a hundred kilobytes is not much history — and it is the price of being able to stop between pieces. The decompressor must use the same piece boundaries implicitly, which it does by decoding until each stream ends and starting the next where it stopped.

The piece size, 100 KiB, is the granularity of responsiveness: the callback fires once per piece, so the maximum time between yields is however long one piece takes.

**Notes** — the name promises multiple threads and the implementation has none. The lock is held for the entire loop, including across every callback — so a callback that tries to compress anything else deadlocks. The honest description is "chunked with a yield hook", and a rebuild should name it that.
