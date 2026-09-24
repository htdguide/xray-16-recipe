# src/xrCore/Compression/PPMdType.h

> The coder's compile-time configuration: its signature word, its order ceiling, and the byte-at-a-time stream it is wired to.

**Needs** — [`compression_ppmd_stream.h`](compression_ppmd_stream.h.md)
**Used by** — [`Coder.hpp`](Coder.hpp.md) · [`Model.cpp`](Model.cpp.md) · [`PPMd.h`](PPMd.h.md) · [`SubAlloc.hpp`](SubAlloc.hpp.md) · [`compression_ppmd_stream.h`](compression_ppmd_stream.h.md)
**Tier floor** — T2 as written: four constants and a choice of stream type. It only looks lower because it is spelled in preprocessor conditionals.

## Purpose

The adopted coder was written to be retargeted by editing one header — pick an environment, pick a stream type, pick a calling convention. Almost all of that is incidental: a rebuild picks its stream type once and deletes the switchboard. Two things in it are not incidental and are the reason this file gets a page.

## State

```text
signature   = 0x84ACAF8F    # a fixed word, stamped into the dummy escape context
variant     = 'I'           # the algorithm generation; never read at run time
MAX_ORDER   = 16            # the highest model order the fixed-size scratch arrays admit
```

**Invariants** — `MAX_ORDER` is a hard ceiling, not advice. The model walks up the suffix chain collecting states into a stack sized exactly `MAX_ORDER`, so an order above it overruns that stack. The engine asks for order 8 (see [`ppmd_compressor.cpp`](ppmd_compressor.cpp.md)), leaving a wide margin.

The signature word is written into the *dummy* escape context — the one used when a context already holds all 256 symbols and so has no escape to estimate. Nothing ever reads it back. It is there so that a memory dump of the pool shows an obvious marker where that context lives, and a rebuild may use any recognizable constant or none at all.

## Stream binding

The coder does all its input and output through four named operations — get a byte from the encoded side, put a byte to it, and the same pair for the decoded side. This header is where those four names are bound to the engine's memory cursor ([`compression_ppmd_stream.h`](compression_ppmd_stream.h.md)), replacing the original's file handle.

**Notes** — the header is included twice in a row, with an assertion macro suppressed for the first inclusion and restored for the second. That is a workaround for the engine's assertion macro colliding with a name inside the adopted code, and it disappears in any rebuild that does not have a global preprocessor.

The prefetch switch, the packed-structure attribute and the calling-convention names are all artifacts of 2001-era compilers. What survives is the *intent* behind the prefetch: the model dereferences a pointer it is about to walk, several steps early, because the context tree is a pointer chase over a pool far larger than cache. A rebuild on a machine where that still matters should keep the touch; on one where it does not, deleting it changes nothing about the output bytes.
