# src/xrCore/Math — two pseudo-random generators and why there are two

Part of chapter 6, [`src/xrCore`](../README.md). Three files. The engine's *general* random
generator is not here — it is [`../_random.h`](../_random.h.md) — and neither is any of the
vector or matrix arithmetic, which is chapter 3.

## What this module is responsible for

Two specialized generators, each existing because the general one was wrong for a specific
caller.

The **32-bit generator** is a one-word linear congruential sequence whose range reduction is
a widening multiply rather than a remainder. It is the generator the archive obfuscator
drives, which makes its exact arithmetic part of a frozen format
([`../Crypto/trivial_encryptor.cpp`](../Crypto/trivial_encryptor.cpp.md)).

The **16-bit per-thread generator** exists for a different reason entirely: it must be
callable from inside a work-stealing parallel loop, where taking a lock or touching a shared
word would serialize the whole loop. One multiply-add per draw, sixteen bits out, state held
per thread.

## Where it sits

It rests on nothing but the scalar types. The 32-bit generator is consumed by the archive
obfuscator; the per-thread one by the renderer's sampling code and by any parallel loop that
needs jitter. Neither is the generator the game simulation uses — see
[`../_random.h`](../_random.h.md) for that one, and note that *its* determinism is what
[`SYSTEM-REQUIREMENTS.md` §6](../../../SYSTEM-REQUIREMENTS.md#6-conformance) asks for.

## The load-bearing ideas

**Range reduction by widening multiply, not remainder.** A draw in `[0, n)` is the top half
of the 64-bit product of the state and `n`. This is faster than a division and, unlike a
remainder, does not bias toward small values when `n` does not divide the period. It is also
*different* from a remainder, so the two cannot be interchanged in any generator whose output
is part of a format.

**Per-thread state is the point, not an optimization.** A generator shared across a parallel
loop is either wrong (torn state) or slow (a lock, or an atomic in the inner loop). The
per-thread one is neither, at the cost of being unreproducible across runs — which is exactly
why it must never be used for anything the simulation depends on.

**Seeding is the only part that cannot be inline.** The per-thread generator's whole body is
a multiply-add; the one thing it cannot do without reaching outside is obtain an initial
value from the world. That is the entire reason it has a `.cpp` at all.

## The twins

| File | Role |
|---|---|
| [`Random32.hpp`](Random32.hpp.md) | **The 32-bit generator**: one multiply-add, widening-multiply range reduction. Its constants are frozen by the archive obfuscator. |
| [`fast_lc16.hpp`](fast_lc16.hpp.md) | **The per-thread 16-bit generator**: cheap enough to call inside a work-stealing loop. Substantive. |
| [`fast_lc16.cpp`](fast_lc16.cpp.md) | The one thing the generator cannot do inline: get a seed from the world. |
