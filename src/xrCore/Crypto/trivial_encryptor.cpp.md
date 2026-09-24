# src/xrCore/Crypto/trivial_encryptor.cpp

> Obfuscates the archive directory of the later game generations: a fixed byte permutation followed by a keystream from a seeded generator. **Frozen** — the shipped archives are encoded with it.

**Needs** — [`trivial_encryptor.h`](trivial_encryptor.h.md) · [`../Math/Random32.hpp`](../Math/Random32.hpp.md) · [`../LocatorAPI.cpp`](../LocatorAPI.cpp.md)
**Used by** — [`trivial_encryptor.h`](trivial_encryptor.h.md)
**Tier floor** — T2: two 256-byte tables and a multiply-add generator. The exact 32-bit wraparound of that generator is load-bearing.

## Purpose

The game's later archive generations store their directory chunk obfuscated. This is not security and the file's own name says so — the keys are compiled into the binary and the algorithm is a substitution followed by a keystream. Its purpose was to stop casual repacking, and the engine's purpose in reproducing it is simply that the shipped data is in this form and must be read.

Two keys shipped, for the Russian and the international releases of the same game. The engine does not know which it has: [`../LocatorAPI.cpp`](../LocatorAPI.cpp.md) decodes the directory with the worldwide key, attempts to decompress it, and on failure re-encodes and retries with the Russian key. **Decompression success is the only oracle for which key is right** — there is no marker in the archive.

## State

```text
RECORD Key
  table_iterations : int     # how many swaps to shuffle the alphabet with
  table_seed       : int     # the generator seed for the shuffle
  encrypt_seed     : int     # the generator seed for the keystream

RECORD Encryptor
  key            : Key
  current        : key_flag          # which key the tables currently reflect
  alphabet       : bytes[256]        # the permutation
  alphabet_back  : bytes[256]        # its inverse
  # invariant: alphabet_back[alphabet[i]] == i for every i
```

### The two keys — verbatim

| | iterations | table seed | keystream seed |
|---|---|---|---|
| **worldwide** | 1024 | 6011979 | 24031979 |
| **russian** | 2048 | 20091958 | 20031955 |

These are dates written as decimal digits — 6 January 1979, 24 March 1979, 20 September 1958, 20 March 1955 — which is the only thing about them that is explicable. They are otherwise arbitrary and must be copied exactly; every digit is part of the format.

**Invariants** — the instance is shared process-wide and **mutates on a key change**, rebuilding both tables. Two threads decoding archives with different keys corrupt each other's tables. Nothing in the engine does this today because archive mounting is single-threaded, and a rebuild should make the tables immutable per key — build both at startup, select by argument — at a cost of 512 bytes.

## `initialize`

**Contract** — select the key, then build the permutation and its inverse. Deterministic: the same key always yields the same tables. Allocates nothing beyond the fixed arrays. Aborts on an unrecognized key.

```text
FUNCTION build_tables(which)
  key := the named key
  FOR i IN 0 .. 255
    alphabet[i] := i                          # start from the identity
  gen := generator seeded with key.table_seed
  REPEAT key.table_iterations TIMES
    j := gen.next(256)
    k := gen.next(256)
    WHILE k == j                              # never a self-swap
      k := gen.next(256)
    swap alphabet[j] and alphabet[k]
  FOR i IN 0 .. 255
    alphabet_back[alphabet[i]] := i           # invert
```

**Invariants** — the retry loop that rejects `k == j` consumes generator draws, so it is part of the sequence: an implementation that instead skips the swap without redrawing produces a *different permutation* and decodes the shipped archives to garbage. The loop cannot spin forever in practice, but nothing bounds it.

**Notes** — the two iteration counts, 1024 and 2048, do nothing more than make the two keys produce different permutations; either is far past the point where the shuffle is thoroughly mixed. Shuffling by repeated random transpositions rather than by a single pass is not a good shuffle, but the quality is irrelevant — what matters is that it is *reproducible*.

## `encode`

**Contract** — substitute each input byte through the permutation, then combine it with the next keystream byte. Source and destination may be the same buffer; the transform is strictly positional so in-place is safe. Rebuilds the tables first if the requested key is not the one currently installed. Does not allocate, does not fail, is not thread-safe.

```text
FUNCTION encode(src, n, dst, which)
  IF which IS NOT current THEN build_tables(which)
  gen := generator seeded with key.encrypt_seed
  FOR i IN 0 .. n-1
    dst[i] := alphabet[src[i]] XOR (gen.next(256) AND 0xFF)
```

## `decode`

**Contract** — the exact inverse: strip the keystream byte, then map back through the inverse permutation. Same in-place safety, same table rebuild, same lack of thread-safety.

```text
FUNCTION decode(src, n, dst, which)
  IF which IS NOT current THEN build_tables(which)
  gen := generator seeded with key.encrypt_seed
  FOR i IN 0 .. n-1
    dst[i] := alphabet_back[src[i] XOR (gen.next(256) AND 0xFF)]
```

**Invariants** — the keystream restarts from the key's seed at **every call**, so it is positional, not streaming: byte *i* of any buffer always gets the same keystream byte. Decoding a buffer in two calls therefore gives the wrong answer for the second half. Every caller in the engine passes whole buffers.

Reusing one keystream for every archive is the reason this is obfuscation and not encryption, and is worth stating plainly so that a rebuild does not mistake it for something to preserve on security grounds.

## The keystream generator

The draws come from the engine's small 32-bit generator ([`../Math/Random32.hpp`](../Math/Random32.hpp.md)), and its exact arithmetic is part of this format:

```text
state := (0x08088405 * state + 1)      # 32-bit, WRAPS
draw  := (state * range) >> 32         # 64-bit product, high half
```

**Invariants** — the wraparound is relied upon; the multiply for the range must be done at 64 bits and the top half taken, not a modulo. A rebuild that substitutes any other generator, or the same generator with a modulo range reduction, decodes every shipped archive to garbage.

Only the low eight bits of each draw are used for the keystream, even though the draw was already reduced to the range 0..255 — so the mask is redundant. It is harmless and worth keeping only as a reminder that the draw is a byte.
