# src/xrCore/crc32.cpp

> The standard CRC-32 over a byte range, plus a path variant that ignores separators — the hash under the string interner, the archive checksums and the filesystem's path lookup.

**Needs** — [`xr_types.h`](xr_types.h.md) · [Seam: Compression](../../SYSTEM-REQUIREMENTS.md#seam-compression)
**Used by** — [`SkeletonMotions.cpp`](Animation/SkeletonMotions.cpp.md) · [`FileCRC32.cpp`](FileCRC32.cpp.md) · [`_std_extensions.h`](_std_extensions.h.md) · [`xrsharedmem.cpp`](xrsharedmem.cpp.md) · [`xrstring.cpp`](xrstring.cpp.md)
**Tier floor** — T1: it is a frozen bit-exact function over bytes whose output appears in shipped archive headers. Any tier can compute it; the byte-for-byte result is what is load-bearing.

## Purpose

One checksum serves three unrelated jobs: it keys the string interner's buckets, it verifies mounted archives against the checksums recorded in them, and — in its path-ignoring variant — it keys the virtual filesystem's file table. Because the second use compares against numbers written by a 2007 engine, the polynomial and the bit order are **frozen**.

## State

Stateless. One 256-entry lookup table of 32-bit values, derived at build time from the polynomial and never mutated.

## `crc32`

**Contract** — Takes a byte range and returns a 32-bit checksum. Pure, allocation-free, thread-safe. An overload takes a previous result and continues from it, which is what lets a large file be checksummed in chunks without buffering it.

```text
FUNCTION crc32(bytes, length) -> int (32-bit)
  # Standard reflected CRC-32: polynomial 0x04C11DB7, initial value all
  # ones, table-driven a byte at a time from the low end, final complement.
  # This is the PKZip / Ethernet variant; the stored checksums in the
  # shipped archives were produced by it and nothing here may change.
  acc = 0xFFFFFFFF
  FOR EACH b IN bytes
    acc = (acc SHIFTED RIGHT 8) XOR table[(acc AND 0xFF) XOR b]
  RETURN acc XOR 0xFFFFFFFF

FUNCTION crc32_continued(bytes, length, previous) -> int (32-bit)
  # Undo the final complement of the previous result, run, complement again.
  acc = 0xFFFFFFFF XOR previous
  ... same loop ...
  RETURN acc XOR 0xFFFFFFFF
```

**Notes** — The table is built by reflecting each byte value into the high end, running eight polynomial steps, and reflecting the 32-bit result back. Building it that way rather than writing the reflected polynomial directly is a transcription of the algorithm's textbook form; a rebuild may hard-code the table or derive it any way it likes, as long as the values match.

## `path_crc32`

**Contract** — Same checksum, but every forward slash and every platform path separator in the input is **skipped entirely** — not substituted, skipped, so the accumulator is not advanced for those bytes.

**Invariants** — Two paths that differ only in which separator they use hash identically. This is what makes the virtual filesystem's lookup table insensitive to the separator the caller wrote, given that the game data mixes both. It is *not* case-insensitive — the caller lowercases first.

**Notes** — Skipping rather than normalizing means `a/b` and `ab` also collide. That is accepted: the table stores the full path in each entry and compares it on a hit, so a collision costs a comparison, not a wrong answer.
