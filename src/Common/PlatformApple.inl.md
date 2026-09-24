# src/Common/PlatformApple.inl

> The macOS fill-in — byte for byte the Linux fill-in, differing only in which system headers supply the same declarations.

**Needs** — [`Platform.hpp`](Platform.hpp.md) · [`PlatformLinux.inl`](PlatformLinux.inl.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`Platform.hpp`](Platform.hpp.md)
**Tier floor** — T1: same reasons as the Linux fill-in it duplicates.

## Purpose

This file is a copy of [`PlatformLinux.inl`](PlatformLinux.inl.md) with three header
substitutions. Every type, every conversion, every bounded string operation and the
complete-read loop are identical, and the substance is documented there — read that twin,
not this one.

The split is arbitrary and a rebuilder should treat it as such: there is one POSIX fill-in
in this engine, physically stored three times. Maintaining it as three copies means a fix
to one is a fix to one.

## State

Stateless — see [`PlatformLinux.inl`](PlatformLinux.inl.md).

## Differences from the Linux fill-in

**Contract** — three, all of them "the same declaration lives in a different header on this
system":

- the maximum path length comes from the system's own limits header rather than the generic
  one;
- the locale type used by the case-insensitive comparisons comes from an extended-locale
  header;
- stack allocation needs no separate header, and the file-control header needs no guard.

**Notes** — the file still defines the marker that names this platform "Linux" to the dead
matchmaking module, and still routes text-integer conversion through the windowing library.
Both are inherited copies rather than decisions about this platform.
