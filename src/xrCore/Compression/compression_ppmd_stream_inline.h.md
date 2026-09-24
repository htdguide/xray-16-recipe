# src/xrCore/Compression/compression_ppmd_stream_inline.h

> The byte cursor's operations, and the asymmetry between reading and writing that the coder depends on.

**Needs** — [`compression_ppmd_stream.h`](compression_ppmd_stream.h.md)
**Used by** — [`compression_ppmd_stream.h`](compression_ppmd_stream.h.md)
**Tier floor** — T2.

## Purpose

Defines the cursor declared in [`compression_ppmd_stream.h`](compression_ppmd_stream.h.md). Small, but two of its five operations encode a decision the coder depends on.

## State

```text
RECORD Cursor
  buffer : bytes          # borrowed, never owned
  length : int
  offset : int            # 0 .. length
```

## Operations

```text
FUNCTION put(c, byte)
  # Writing past the end is a PROGRAMMING ERROR, checked only in a diagnostic
  # build. The caller is expected to have sized the output buffer from the
  # worst case before starting.
  c.buffer[c.offset] := byte
  c.offset := c.offset + 1

FUNCTION get(c) -> int
  # Reading past the end is a NORMAL CONDITION and returns the end-of-input
  # marker. That is how the encoder learns the input is exhausted: it reads
  # until the marker comes back.
  IF c.offset >= c.length THEN RETURN end_of_input
  byte := c.buffer[c.offset]
  c.offset := c.offset + 1
  RETURN byte

FUNCTION rewind(c)   c.offset := 0
FUNCTION tell(c)     RETURN c.offset
```

**Invariants** — the asymmetry is the point and is not an oversight: the coder terminates its main loop on the read side returning the marker, and has no mechanism at all for a full output. A rebuild must therefore guarantee the output buffer is large enough before starting, which is why the caller sizes it from the worst case.

The end-of-input marker is distinguishable from a real byte only because the return is wider than a byte. A rebuild in a language with a proper optional should use one.

**Notes** — `rewind` exists for one caller: replaying a pre-trained model, which is read from the start for each compression. See [`ppmd_compressor.cpp`](ppmd_compressor.cpp.md).
