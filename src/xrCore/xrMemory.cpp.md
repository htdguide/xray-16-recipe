# src/xrCore/xrMemory.cpp

> The allocator seam: one routing point every allocation in the engine passes through, three interchangeable fillings behind it, and the process-memory queries the out-of-memory path reports.

**Needs** — [`xrMemory.h`](xrMemory.h.md) · [`xrstring.h`](xrstring.h.md) · [`xrsharedmem.h`](xrsharedmem.h.md) · [`log.h`](log.h.md) · [`Memory/xrMemory_align.h`](Memory/xrMemory_align.h.md) · [Seam: Allocator](../../SYSTEM-REQUIREMENTS.md#seam-allocator) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`xrMemory.h`](xrMemory.h.md)
**Tier floor** — T1: it replaces the language's own allocation operators wholesale and hands out alignment guarantees that graphics and audio buffers are built on.

## Purpose

Every container, every object and every raw buffer in the engine is allocated through one object, so that the allocator can be swapped at build time without touching a call site. That indirection is the whole point: the engine has been built against its own pooled allocator, the system allocator, and a third-party one, and the choice matters mostly at 32 bits where the address space is the constraint.

## State

```text
RECORD Allocator                 # one process-global instance
  # No fields. The state lives in whichever filling is compiled in.
```

The global instance must be constructible before anything else runs and must not depend on any other global — it is used by the constructors of other globals. Two things it *owns* rather than holds: the string interner and the shared-blob interner are created by its initialization step and destroyed by its teardown, in that order reversed, because both allocate through it.

## `initialize` / `destroy`

**Contract** — Initialization creates the string interner, marks it usable (a flag the failure path reads, so that a failure *before* this point does not try to report interner statistics), then creates the shared-blob interner. Teardown destroys them in the reverse order. Neither is reentrant; both run once, from the core's own bring-up.

**Invariants** — The "interner is usable" flag exists because the failure path reports interner statistics, and a failure during early bring-up would otherwise dereference an interner that does not exist yet.

## Allocation surface

**Contract** — Six operations, each in a plain and an aligned form, plus a non-throwing variant of allocation:

| Operation | Contract |
|---|---|
| allocate(size) | returns a block of at least `size` bytes, suitably aligned for any ordinary object; throws or invokes the out-of-memory path on failure |
| allocate(size, alignment) | as above with an explicit alignment; the caller must free it with the matching alignment |
| allocate non-throwing | returns nothing on failure instead of entering the failure path |
| reallocate(block, size) | grows or shrinks, preserving contents up to the smaller size; may move |
| free(block) | releases; releasing nothing is legal |
| free(block, alignment) | releases a block that was allocated with that alignment |

**Invariants** — Alignment is not remembered by the allocator. A block allocated with an explicit alignment **must** be freed with the same alignment, on every filling. This is the one rule that makes the seam swappable, and it is why the language's aligned and plain deallocation operators are both routed and kept distinct.

## Small-block path

**Contract** — A separate allocate/free pair for blocks no larger than a fixed small-size threshold. Non-throwing, and free takes no size. The threshold is `128 * pointer_size` — 1024 bytes at 64-bit pointers.

**Invariants** — A block from the small path **must** be freed on the small path, never on the general one. Nothing checks this.

**Notes** — The threshold is chosen to match the third-party allocator's own small-object fast path, and the build asserts that it does not exceed it. That assertion is the real content: the constant is not a tuning decision of this engine, it is a constraint imported from whichever filling is in use. A rebuild whose allocator has no such fast path can make the small path a synonym for the general one and delete the distinction — but must then still keep allocation and release paired, because a filling that *does* have one may be swapped back in.

A scoped helper picks the small path or the general one by size at construction, remembers which, and releases correspondingly. That is the only safe way to use the split, and a rebuild should make it the *only* way.

## The three fillings

**Contract** — Exactly one is compiled in; the build fails if none is chosen. What each provides:

- **Third-party pooled allocator** — the default for shipping builds. Supplies aligned allocation and reallocation, sized free, and the small-object fast path natively.
- **Engine's own aligned wrapper over the system allocator** — the system allocator plus hand-written alignment adjustment.
- **Plain system allocator** — used for debug builds, where the platform's own heap diagnostics are worth more than allocator speed. In non-debug configurations this filling pads **every allocation with eight extra bytes**, deliberately, to mask small overruns that would otherwise corrupt the next block; in debug configurations the padding is removed so that those overruns are *found*.

**Notes** — The eight-byte tail is the clearest example of a decision that must be carried as a decision and not as code: the engine knowingly contains small buffer overruns in old code, and the padding is a shipping-stability measure. A rebuild that cannot reproduce the overruns does not need the padding, and should not add it.

## Global allocation-operator replacement

**Notes** — The language's own allocation and deallocation operators — in every plain, array, aligned, sized and non-throwing spelling — are redirected into this object. That is incidental to the *mechanism* but essential to the *decision*: it means third-party code compiled into the process, and any container the engine did not write, allocates through the same routing point. A rebuild whose language does not allow this must instead route explicitly and accept that foreign code will not participate, which changes what the memory reports below actually measure.

## `memory_usage`

**Contract** — Reports the process's own memory footprint, as one number, from whatever the platform offers: page-file usage on one, peak resident set on the POSIX family, used pages on another. **These are not the same quantity** and are not comparable across platforms — the number is a trend indicator and a crash-report field, nothing more.

## `compact`

**Contract** — Asks the process to give memory back. Sweeps the string interner and the shared-blob interner of everything unreferenced — which is the only part that reliably reclaims anything — and, on request from the command line, asks the operating system to trim the working set.

**Notes** — The file carries a note (in Russian) that heap compaction is no longer worth doing because modern allocators return memory on their own, and that the call is kept only for the case of memory-mapped files needing large contiguous free regions. That is exactly right and a rebuild should not reintroduce it. The interner sweeps, however, are load-bearing: they are what the out-of-memory path actually frees.

## `virtual_memory_info` / `log_virtual_memory_info`

**Contract** — Reports free, reserved and committed memory system-wide, by walking the address space on one platform and reading system counters on the others. Logged at startup and when memory runs out. Yields zeroes on platforms with no equivalent.

## `duplicate_string`

**Contract** — Allocates a copy of a null-terminated byte sequence through the routing point. Exists so that duplicated text is freed by the same allocator that allocated it, which the language's own duplication helper would not guarantee.
