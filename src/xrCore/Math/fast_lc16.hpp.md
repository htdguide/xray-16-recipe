# src/xrCore/Math/fast_lc16.hpp

> A per-thread random source cheap enough to call inside a work-stealing loop: one multiply-add per draw, sixteen bits out.

**Needs** — [`fast_lc16.cpp`](fast_lc16.cpp.md)
**Used by** — [`fast_lc16.cpp`](fast_lc16.cpp.md) · [`TaskManager.cpp`](../Threading/TaskManager.cpp.md)
**Tier floor** — T2: 32-bit wrapping arithmetic and a documented output width. Its reason for existing is a cost budget, not a layout constraint.

## Purpose

The task scheduler picks a victim to steal work from at random, several times per steal attempt, on every worker thread. That call site cannot afford a shared generator (contention), a lock, or a division. This generator answers it: a single wrapping multiply-add, sixteen bits of output taken from the middle of the word, and a seeding rule that guarantees two threads get different streams. It is the class extracted from a threading-building-blocks library, kept because the property it guarantees — long period from an odd increment — is the one the scheduler needs.

It is deliberately *not* the generator the simulation uses. Nothing reproducible may depend on it: the streams differ between runs and between threads by construction.

## State

```text
RECORD FastLC16
  x : int (32-bit, wraps)     # the state
  c : int (32-bit, wraps)     # the increment
# invariant: c is odd. An even increment collapses the period; the draw asserts it.
```

## `FastLC16`

**Contract** — constructible from an explicit 32-bit or 64-bit seed, from the address of any object (so "give this worker its own stream" is one line), or from nothing at all, in which case it takes a seed from the platform's entropy source — see [`fast_lc16.cpp`](fast_lc16.cpp.md). Exposes the minimum and maximum of its output type so it can stand in wherever a standard-library distribution expects a generator. Drawing does not allocate, does not block, and is not safe to share between threads.

```text
FUNCTION seed_from(value: int (32-bit))
  c <- (value OR 1) * 0xba5703f5     # forcing bit 0 keeps c odd; the prime
                                     # multiply decorrelates threads whose
                                     # seeds differ only in the low bits
  x <- c XOR (value >> 1)            # so the very first draw is already stirred

FUNCTION seed_from(value: int (64-bit))
  seed_from(low 32 bits of ((value >> 32) + value))

FUNCTION draw() -> int (16-bit)
  result <- bits 16..31 of x         # the high half: better distributed than the low
  x <- x * 0x9e3779b1 + c            # 32-bit, wrapping
  RETURN result
```

**Notes** — the multiplier is the 32-bit golden-ratio constant, a large prime chosen for its bit-mixing rather than for any statistical guarantee. The result is read *before* the step, so a freshly seeded generator's first output already reflects the seeding stir; that ordering is why `x` is seeded from a shuffled value rather than from the raw seed.

Only the top half of the word is returned. Low bits of a linear congruential sequence have short periods — bit 0 alternates, bit 1 has period 4, and so on — so taking the low half would be visibly non-random at exactly the small ranges the scheduler asks for.

Seeding from an object's address is a legitimate use here and would be a bug anywhere reproducibility mattered: it makes the stream depend on the memory layout of the run.
