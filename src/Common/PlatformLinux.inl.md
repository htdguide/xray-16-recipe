# src/Common/PlatformLinux.inl

> The Linux (and Haiku) fill-in — the largest of the four, because it must emulate the Windows vocabulary the engine was written in on top of POSIX.

**Needs** — [`Platform.hpp`](Platform.hpp.md) · [`Compiler.inl`](Compiler.inl.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services) · [Seam: Windowing and input](../../SYSTEM-REQUIREMENTS.md#seam-windowing-and-input)
**Used by** — [`Platform.hpp`](Platform.hpp.md) · [`PlatformApple.inl`](PlatformApple.inl.md) · [`PlatformBSD.inl`](PlatformBSD.inl.md)
**Tier floor** — T1: it fixes the exact widths of types that cross the operating-system and driver boundary, and reimplements bounded string operations byte by byte.

## Purpose

The engine's source speaks Windows: it names Windows integer types, calls Windows string
functions, and writes paths with Windows separators. Porting it did not mean rewriting that
vocabulary — it meant supplying the vocabulary on the other side. This file is that supply,
and reading it is the fastest way to see exactly how wide the Windows surface the engine
actually depends on is: about a dozen types, about thirty functions, and two behaviours
(path separators and case folding) that are genuinely different rather than merely renamed.

A rebuild deletes almost all of this. What it must *not* delete are the four sections below
marked as real decisions, because those describe behaviour, not naming.

## State

```text
RECORD PlatformTypes              # widths are load-bearing: these cross the
                                  # graphics, audio and physics boundaries
  boolean_flag  : int (32-bit)
  word          : int (16-bit, unsigned)
  double_word   : int (32-bit, unsigned)
  long_value    : int (32-bit, signed)     # note: 32-bit even on 64-bit targets
  pointer_sized : int (64-bit on x64/arm64/e2k, 32-bit otherwise)
  result_code   : int (signed, native word)   # negative means failure
  opaque_handle : pointer
```

**Invariants**

- `long_value` is 32 bits even where the platform's own `long` is 64. It appears in
  structures compared against driver expectations, so widening it would silently change
  layouts.
- The pointer-sized integers are chosen from the architecture classification, not from the
  compiler's `long`, because the two disagree on exactly the targets the engine reaches.
- The message-parameter types differ on one architecture (64-bit PowerPC takes the signed
  form) — a detail with no discoverable justification in the source beyond matching what
  that target's own convention does.
- A result code is a failure exactly when it is negative. Two named successes exist, one
  meaning "done" and one meaning "done, nothing happened".

## Path separator conversion

**Contract** — this is the first of the four real decisions. The engine's canonical separator is
the backslash, because that is what the shipped archives and configuration files contain.
Two conversions exist and both are in-place over a mutable path:

```text
FUNCTION to_host_form(path)         # called immediately before touching a real file
  REPLACE every '\' IN path WITH '/'

FUNCTION to_canonical_form(path)    # called on names read back from the real filesystem
  REPLACE every '/' IN path WITH '\'
```

**Invariants** — a path is in canonical form everywhere inside the engine and in host form only
at the moment of a system call. Every path-taking system call this file wraps — delete,
remove directory — performs the conversion itself on a private copy, so callers never have
to remember.

## Filesystem name folding

**Contract** — the second real decision, and the inverse of the Windows answer. On a
case-sensitive filesystem the *stored* name is folded to lower case and the lookup key is
left alone, because the engine's own index — not the filesystem — is what has to match
inconsistently-cased references in the game data.

**Invariants** — the fold is ASCII-only. A locale-aware fold changes which files match and is
explicitly wrong here; see
[§4 Platform assumptions](../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions).

## Complete read

**Contract** — the third real decision. Read exactly the requested number of bytes from a file
descriptor into a buffer, or fewer only at end of file, or report failure. This exists
because on this platform a single read may legitimately return less than asked for with no
error and no end of file, which every other platform the engine targets does not do — and
every caller in the engine was written assuming a complete read.

```text
FUNCTION read_fully(handle, buffer, count) -> int   # bytes read, or -1
  total <- 0
  WHILE total < count
    n <- read(handle, buffer at offset total, count - total)
    IF n = -1
      RETURN -1                    # a genuine error, reported as-is
    IF n = 0
      RETURN total                 # end of file; a short result here is truthful
    total <- total + n
  RETURN total
```

**Notes** — the end-of-file arm is what makes the loop terminate; without it a truncated file
spins forever. A rebuild whose runtime already guarantees complete reads deletes this
entirely, but should confirm the guarantee rather than assume it.

## Bounded string operations

**Contract** — the fourth group, and the only one where this file writes real algorithms: bounded
copy, bounded copy of at most n characters, bounded append, and bounded append of at most n
characters. Each takes a destination and its capacity, and each reports one of three
outcomes — success, invalid argument, or "would not fit".

**Invariants** — on any failure the destination is left as an empty string rather than a partial
copy. That is the load-bearing part: a truncated string that still looks like a path or a
section name is far more dangerous than an obviously empty one, so the failure is made
visible at the next use.

**Notes** — the return values deliberately match the original Windows functions' error numbering,
because call sites test them. One of these (the bounded append of n characters) writes its
terminator inside the inner loop in a way that only works because the outer loop has
already found the existing terminator; it is fragile and any rebuild should write these
four from the contract above rather than transliterate them.

## Path splitting

**Contract** — split a path into drive, directory, base name and extension, writing each into a
caller-supplied buffer, any of which may be omitted. The drive is always empty on this
platform. The directory is everything up to and including the last separator of *either*
kind — the split accepts both, because it runs on paths that may not have been converted
yet. The extension begins at the *first* dot of the remaining name, not the last.

**Notes** — first-dot rather than last-dot is a real behavioural difference from the platform
this emulates, and it changes the answer for names like `level.geom.1`. Nothing in the
source says whether that is intentional; treat it as a hazard to check against shipped
asset names rather than as a decision to reproduce blindly.

## Absent-service stubs

**Contract** — several Windows process services have no meaning here and are given stubs that
answer neutrally: the last-error query always reports "no error", the exception-code query
always reports zero, and the structured-exception record type is empty.

**Notes** — these are not fallbacks, they are amputations: any engine code that branches on the
last error is, on this platform, taking the no-error branch unconditionally. A rebuild
should find those call sites rather than reproduce the stubs — a stub that always says
"fine" is the least debuggable shape a missing capability can take.

## Notes

The debug-output function writes to the standard error stream, which is how a developer
sees engine logging under a debugger on this platform.

Thread yield is mapped onto the scheduler's yield and reports whether it succeeded.

Text-integer conversion is deliberately routed through the windowing library rather than
the C library, because the C library's version is not available everywhere the engine
builds; if that library's header is absent the build fails with a message saying so rather
than silently losing the function. That is the right shape for a missing dependency and
contrasts with the stubs above.
