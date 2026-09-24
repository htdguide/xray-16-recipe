# src/xrCore/string_concatenations.cpp

> The failure side of joining: the oversized-join report, and the stack-exhaustion probe.

**Needs** — [`string_concatenations.h`](string_concatenations.h.md) · [`string_concatenations_inline.h`](string_concatenations_inline.h.md) · [`xrDebug.h`](xrDebug.h.md) · [`xrMemory.h`](xrMemory.h.md)
**Used by** — [`string_concatenations.h`](string_concatenations.h.md) · [`string_concatenations_inline.h`](string_concatenations_inline.h.md)
**Tier floor** — T1: it probes the stack for available space inside a fault handler and resets the guard page afterwards.

## Purpose

Two things that cannot be inline: the report produced when a stack-allocating join exceeds its ceiling, and the probe that keeps such a join from taking the process down when the stack is nearly full.

## State

Stateless.

## Oversized-join report

**Contract** — Takes the index of the part that crossed the ceiling, the part count, and the parts, and fails fatally with a message naming the index and listing every part. Each part is quoted in brackets, one per line, **truncated to 1024 bytes** — because the usual cause is an unterminated string and printing it whole would be unbounded.

```text
FUNCTION report_overflow(index, count, parts) -> void
  build a bracketed, line-separated listing of the parts,
    each cut at 1024 bytes
  fatal("buffer overflow: cannot concatenate strings(index):\n" + listing)
```

**Notes** — In a shipping build the report is compiled out and the overflowing copy simply backs off by one byte instead — the join truncates silently rather than killing a player's session. That asymmetry is deliberate and is the same policy as the assertion classes: diagnose in development, degrade in shipping.

## Stack probe

**Contract** — Attempts to take a given number of stack bytes inside a fault handler. On a stack-overflow fault, resets the guard page so the stack is usable again and returns normally; the caller then proceeds and its own allocation fails the same way, but recoverably. On every other fault, the fault is passed on. A no-op on platforms without a guard-page reset.

**Invariants** — The guard-page reset must happen **after** the handler has been entered, never inside the filter that selects it, because at filter time the stack has not yet been unwound. The source states this explicitly and it is the one subtlety of the routine.

**Notes** — Portable placeholders exist so the rest of the code compiles where none of this applies. A rebuild on a tier that grows stacks dynamically, or that has no stack-allocated strings, removes the whole file and keeps only the size ceiling from [`string_concatenations_inline.h`](string_concatenations_inline.h.md).
