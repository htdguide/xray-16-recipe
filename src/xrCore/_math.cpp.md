# src/xrCore/_math.cpp

> Process and per-thread numeric bring-up: detect the CPU's vector instruction sets, seed the global random generator, build the compressed-normal table, and put every thread's floating-point unit into flush-to-zero mode.

**Needs** — [`_math.h`](_math.h.md) · [`_matrix.h`](_matrix.h.md) · [`_random.h`](_random.h.md) · [`_compressed_normal.h`](_compressed_normal.h.md) · [`xrDebug.h`](xrDebug.h.md) · [`log.h`](log.h.md) · [`_std_extensions.h`](_std_extensions.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)

**Used by** — [`_math.h`](_math.h.md) · [`_random.h`](_random.h.md)

**Tier floor** — T1: it reads CPU capability flags and writes a control register that changes how the floating-point unit rounds — neither has a portable expression above this tier.

## Purpose

Three globals and two entry points. The globals are the identity transform, the engine-wide
random generator, and a record of what the processor can do; the entry points are
"the process is starting" and "a thread is starting". Everything numeric that must be true
before the first frame is established here.

The file is a separate file mostly because the things in it have nothing to do with each
other except *when* they happen. A rebuild is free to fold this into its startup sequence,
provided the ordering constraints below survive.

## State

```text
RECORD CpuCapabilities            # read once at load, never written again
  has_sse       : bool
  has_sse2      : bool
  has_sse4_2    : bool
  has_avx       : bool
  has_avx2      : bool
  has_avx512f   : bool
  counter_freq  : int (64-bit)    # ticks per second of the monotonic counter
  counter_reads : int (32-bit, wraps)   # how many times the counter was read

GLOBAL identity_transform : Matrix4x4    # a constant, but built at startup not compiled in
GLOBAL random             : RandomGenerator
```

**Invariants**

- The capability flags and the counter frequency are established **before** any engine code
  runs — they are initialized at module load, not inside the startup function. A rebuild
  that queries them lazily must make sure nothing reads them first, because the math layer
  branches on `has_sse` in per-frame code.
- `counter_reads` is a diagnostic counter, incremented on every clock read and reset by the
  on-screen statistics overlay once per frame. It is deliberately unsynchronized: it is read
  from every thread and is only ever looked at by a human, so a lost increment costs
  nothing. A rebuild that makes it atomic pays for it in the hottest non-rendering path in
  the engine.
- The identity transform is a mutable global that everything reads and nothing is supposed
  to write. That it is filled at startup rather than being a compile-time constant is an
  artifact; a rebuild makes it a constant.

## `QPC` — the monotonic tick counter

**Contract** — Returns the platform's high-resolution monotonic counter, in ticks whose
rate is `counter_freq`. Never blocks, never allocates, never fails, and increments the
diagnostic read counter as a side effect. This is the clock the frame loop, the profiler and
the scheduler all measure against, and the platform assumptions require that it not step
backwards.

## `GetTicks` — the millisecond clock

**Contract** — Milliseconds since process start, as a 32-bit value that wraps after about
49 days. Used where millisecond resolution is enough and the cost of the high-resolution
read is not wanted.

**Notes** — The wrap is real and is not handled anywhere. Nothing in a session lasts 49
days, which is the reason it is not handled and is the assumption a rebuild inherits.

## `_initialize_cpu` — process bring-up

**Contract** — Called once, early, from the core module's startup. Logs the processor's
feature set and thread count, then establishes the three numeric globals. Blocks for as
long as the logging takes; allocates nothing beyond a stack buffer.

```text
FUNCTION initialize_process_math() -> void
  # Build a human-readable feature list for the log. This is diagnostics only:
  # nothing branches on it, and every crash report carries it, which is why it
  # is the first thing written.
  features = comma-joined names of every supported instruction set
             (the x86 ladder from the timestamp counter through AVX-512,
              then the non-x86 vector sets: AltiVec, ARM SIMD, NEON, LSX, LASX)
  LOG "CPU features: " + features
  LOG "CPU threads: " + hardware_thread_count

  identity_transform.set_identity()

  # Seed from the low 32 bits of the monotonic counter. The generator's state
  # is 32 bits, so anything wider is thrown away; the counter is the only
  # entropy the engine has at this point.
  random.seed(low 32 bits of QPC())

  build_compressed_normal_table()

  initialize_thread_math()          # the calling thread counts as a thread
```

**Invariants** — Ordering is load-bearing in one place only: the compressed-normal table
must be built before anything decodes a normal, which in practice means before the first
model loads. The rest of the sequence is free.

**Notes** — Seeding from the clock means **runs are not reproducible by default**. The
determinism criterion in the conformance list ("the same level, the same input sequence and
the same seed") is therefore conditional on something re-seeding the generator afterwards;
nothing in this file does. A rebuild that wants replayable sessions must expose the seed.

The feature list is assembled by appending to a fixed-size text buffer with no bound check
beyond the buffer's own size. That is incidental — the problem it solves is "build a
comma-separated list without allocating during startup" — but the buffer is 256 bytes and
the list has a known maximum length well under that.

## `_initialize_cpu_thread` — per-thread bring-up

**Contract** — Must be called once on every thread the engine creates, before that thread
does any floating-point work. Registers the thread with the crash handler and puts its
floating-point control register into flush-to-zero and denormals-are-zero mode. Does not
allocate, does not block.

```text
FUNCTION initialize_thread_math() -> void
  register_thread_with_crash_handler()
  IF NOT cpu.has_sse THEN RETURN

  set_flush_to_zero(on)              # results that would be subnormal become zero

  IF denormals_are_zero_supported THEN
    TRY
      set_denormals_are_zero(on)     # subnormal *inputs* are treated as zero
    ON FAULT
      denormals_are_zero_supported = false   # remember, and never try again
```

**Invariants**

- Every thread that touches the math layer must run this. The collision module's worker
  threads and the general thread pool both call it on entry; a thread that skips it is not
  *wrong*, but it computes subtly different results near zero from every other thread, which
  breaks the determinism criterion. This is the reason it is a documented step and not an
  implementation detail.
- The `denormals_are_zero_supported` flag is process-wide and deliberately sticky: the
  second instruction is not available on the oldest supported processors, faults there, and
  the fault handler records that fact so no later thread pays the cost of faulting again.
  It is written from multiple threads without synchronization; the only value ever written
  is "false", so a race can only cause a redundant fault.

**Notes** — Two rounding behaviours are being bought here, and a rebuild needs to know that
it is buying *behaviour*, not speed. Flush-to-zero and denormals-are-zero make arithmetic
near zero abrupt rather than gradual, which the engine relies on in two ways: it removes the
hundred-fold slowdown that subnormal arithmetic costs on the processors of the era (a
physics solver that drifts into subnormals otherwise drops frames), and it makes
"very small" and "zero" the same number everywhere, which the epsilon comparisons scattered
through the geometry code are already written to assume.

On architectures with no such control (the ARM and PowerPC targets), both operations are
defined away and the flags simply are not set. The engine runs; it is marginally slower near
zero and marginally different. That divergence is unaddressed in the original and a rebuild
targeting mixed architectures should know it exists.

The fault-catching wrapper is an artifact of two different platform mechanisms for the same
idea — "run this instruction and survive if it is not implemented". A rebuild solves it by
testing the capability flag instead, which is the honest form of the same check.
