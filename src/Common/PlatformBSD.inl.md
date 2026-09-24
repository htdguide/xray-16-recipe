# src/Common/PlatformBSD.inl

> The BSD fill-in — byte for byte the Linux fill-in, differing only in two header inclusions.

**Needs** — [`Platform.hpp`](Platform.hpp.md) · [`PlatformLinux.inl`](PlatformLinux.inl.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`Platform.hpp`](Platform.hpp.md)
**Tier floor** — T1: same reasons as the Linux fill-in it duplicates.

## Purpose

One fill-in serves all four BSD variants; they are distinguished only in the build banner,
never in behaviour. Its content is [`PlatformLinux.inl`](PlatformLinux.inl.md) with two
header substitutions — read that twin for the substance.

As with the Apple fill-in, the split is arbitrary. Three physical copies of one POSIX
fill-in is a maintenance fact, not a design.

## State

Stateless — see [`PlatformLinux.inl`](PlatformLinux.inl.md).

## Differences from the Linux fill-in

**Contract** — two: the standard-definitions header is included explicitly rather than arriving
transitively, and stack allocation needs no separate header. Nothing else differs, including
the file-control header that on Linux is guarded for Haiku's benefit.
