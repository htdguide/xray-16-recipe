# src/xrCore/_std_extensions.cpp

> The local-time stamp used in file names.

**Needs** — [`_std_extensions.h`](_std_extensions.h.md) · [Seam: Threads, atomics and process services](../../SYSTEM-REQUIREMENTS.md#seam-threads-atomics-and-process-services)
**Used by** — [`_std_extensions.h`](_std_extensions.h.md)
**Tier floor** — T3: a formatted local-time conversion.

## Purpose

One function. Everything else declared in [`_std_extensions.h`](_std_extensions.h.md) is inline; this is the part that needs the platform's clock.

## `timestamp`

**Contract** — Writes the current **local** time into a caller-supplied 64-byte buffer as `MM-DD-YY_HH-MM-SS` and returns it. Uses the thread-safe time conversion on every platform. Never fails; truncates at the buffer size, which the format cannot reach.

**Invariants** — Every character in the output is legal in a filename on every supported platform, which is the reason for the format: the value names screenshots, unique log files and crash reports. Local time rather than universal time is deliberate — these names are read by the person sitting at the machine.

**Notes** — Two-digit years and month-before-day mean the names do not sort chronologically. That is a real defect in a directory of a hundred screenshots, and a rebuild should use a sortable form; nothing parses these names back.
