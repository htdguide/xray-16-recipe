# src/xrCore/_math.h

> Declares the processor capability record, the monotonic clock, and the two numeric bring-up entry points.

**Needs** — [`xr_types.h`](xr_types.h.md) · [`_math.cpp`](_math.cpp.md)

**Used by** — [`R_light.h`](../utils/xrLC_Light/R_light.h.md) · [`FTimer.h`](FTimer.h.md) · [`ThreadUtil.h`](Threading/ThreadUtil.h.md) · [`_math.cpp`](_math.cpp.md) · [`vector.h`](vector.h.md)

**Tier floor** — T1: it exposes CPU feature flags and a raw tick counter, which exist only because the layers below are not uniform.

## Purpose

Declares the surface implemented in [`_math.cpp`](_math.cpp.md). Despite the name it holds
no mathematics — the vector, matrix and quaternion vocabulary is in the `_vector*`,
`_matrix*` and `_quaternion` headers, and their algorithms are in the separate math module.
This header is the *numeric environment*: what the processor can do, what time it is, and
the two calls that must happen before anything computes.

## Exported units

**Processor capability flags** — six read-only booleans naming the vector instruction sets
the math layer may use: SSE, SSE2, SSE4.2, AVX, AVX2 and AVX-512 foundation. Only the first
is branched on at run time; the rest are logged and available to callers that want a wider
path.

**`qpc_freq`** — ticks per second of the monotonic counter. Every duration in the engine is
a tick difference divided by this.

**`qpc_counter`** — how many times the counter has been read since the statistics overlay
last reset it. Diagnostics only.

**`QPC`** — read the monotonic counter.

**`GetTicks`** — milliseconds since process start, wrapping at 32 bits.

**`_initialize_cpu`** — process-wide numeric bring-up. Call once, early.

**`_initialize_cpu_thread`** — per-thread numeric bring-up. Call once on **every** thread
before it does floating-point work; see [`_math.cpp`](_math.cpp.md) for why skipping it
changes results rather than merely performance.
