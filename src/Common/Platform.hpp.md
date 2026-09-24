# src/Common/Platform.hpp

> Decides at build time which operating system, processor architecture and compiler the engine is being built for, and pulls in the matching fill-in.

**Needs** — [`Compiler.inl`](Compiler.inl.md) · [`PlatformWindows.inl`](PlatformWindows.inl.md) · [`PlatformLinux.inl`](PlatformLinux.inl.md) · [`PlatformBSD.inl`](PlatformBSD.inl.md) · [`PlatformApple.inl`](PlatformApple.inl.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`Common.hpp`](Common.hpp.md) · [`Compiler.inl`](Compiler.inl.md) · [`PlatformApple.inl`](PlatformApple.inl.md) · [`PlatformBSD.inl`](PlatformBSD.inl.md) · [`PlatformLinux.inl`](PlatformLinux.inl.md) · [`PlatformWindows.inl`](PlatformWindows.inl.md) · [`pch.h`](../utils/mp_balancer/pch.h.md)
**Tier floor** — T1: the whole file exists because the target's services and type widths are not uniform. A language with one standard library across its targets deletes it.

## Purpose

The engine reaches seven operating-system families and eight processor architectures, and
it draws the abstraction at build time rather than run time: exactly one platform fill-in
is compiled, and it defines a common set of names the rest of the engine uses without ever
asking which platform it is on. This file is the switchboard — it classifies the target,
then selects the fill-in.

It also produces the human-readable build banner, which is not decoration: crash reports
and bug reports are triaged by it, so the classification must be visible at runtime.

## State

```text
RECORD BuildTarget                  # every field decided at build time
  platform      : ENUM { windows, android, linux, freebsd, openbsd, netbsd,
                         dragonfly_bsd, apple, haiku }
  is_posix      : bool              # true for everything except windows
  architecture  : ENUM { x86, x64, arm32, arm64, riscv, ppc32, ppc64, e2k }
  compiler      : ENUM { msvc, gcc_compatible }
  static_build  : bool              # renderer and game linked in, or loaded as libraries
  master_gold   : bool              # shipping build: debug instruments compiled out
  configuration : ENUM { debug, mixed, release }
```

**Invariants**

- The classification is total and exclusive: an unrecognized operating system,
  architecture or compiler is a build failure, never a silent default. That is deliberate —
  a wrong guess here produces a program that builds and then misbehaves at the byte level.
- Android is classified as a *variant* of Linux, not a peer: it gets its own name for the
  banner but inherits Linux's entire fill-in. Haiku is the reverse — its own platform
  identity, but it reuses the Linux fill-in with two headers excluded.
- The BSDs share one fill-in and differ only in the banner name.
- `mixed` is a third configuration between debug and release: optimizations on, assertions
  and instrumentation still present.
- `master_gold` is orthogonal to the configuration and suppresses developer-facing
  behaviour (cheats, some logging, the reference-count trace) independently of optimization
  level.

## Target classification

**Contract** — inspects the compiler's own predefined description of the target and yields the
record above. No input, no failure mode except "unsupported", which is reported at build
time.

**Notes** — the architecture split is finer than anything the engine actually branches on. Only
three distinctions earn their keep: pointer width (which fixes the widths of the
Windows-compatibility integer aliases the POSIX fill-ins define), whether the target has
the 4-wide float instruction set natively, and which instruction encodes a debugger trap.
The rest exist to make the banner accurate and to fail loudly on a target nobody has
tested.

Little-endian is assumed rather than detected. Nothing in this file, or anywhere else,
byte-swaps — see [§4 Platform assumptions](../../SYSTEM-REQUIREMENTS.md#4-platform-assumptions).
A big-endian target would classify successfully here and then read every on-disk structure
wrong, so a rebuild that cares should make endianness part of this classification and
refuse what it cannot serve.

## Build banner

**Contract** — composes two strings from the classification: one naming the configuration and
whether it is a shipping build, one naming the platform, pointer width and linkage model.
Both are embedded in the executable and printed at startup.

## Notes

The selection of a fill-in is a one-of-four choice made at build time, and the fourth arm
is an error rather than a fallback: there is no "generic POSIX" path. Adding a platform
means writing its fill-in, which is the honest cost and the reason the list is as short as
it is.
