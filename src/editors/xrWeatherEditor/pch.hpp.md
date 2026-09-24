# src/editors/xrWeatherEditor/pch.hpp

> The library's shared preamble, and the two text conversions every call across the language boundary needs.

**Needs** — [`xrCore/xrCore.h`](../../xrCore/xrCore.h.md) · [`xrEngine/Engine.h`](../../xrEngine/Engine.h.md) · [`xrEngine/device.h`](../../xrEngine/device.h.md) · [`xrSound/Sound.h`](../../xrSound/Sound.h.md) · [Seam: Allocator](../../../SYSTEM-REQUIREMENTS.md#seam-allocator)
**Used by** — [`pch.cpp`](pch.cpp.md)
**Tier floor** — T1: it owns a raw allocation whose release is the caller's responsibility, and a pinning operation that stops a collector moving a buffer mid-call.

## Purpose

Mostly a compilation-speed artifact and therefore mostly incidental. Two things in it are not, and they are why it gets a page: the assertion vocabulary the whole library uses, and the pair of text conversions that every call across the language boundary passes through.

## State

`Stateless.`

## The assertion vocabulary

**Contract** — `VERIFY(condition)` and `NODEFAULT` are checks that exist only in debug builds. In release, `VERIFY` evaluates nothing at all — not even its condition — and `NODEFAULT` becomes a promise to the compiler that a branch is unreachable.

**Notes** — "evaluates nothing at all" is the load-bearing part and it is a trap the recipe must name: **a check with a side effect disappears in a release build.** Several call sites in this library pass a lookup or a narrowing into the check. A rebuild that keeps assertions cheap must either keep them side-effect-free or keep them in release.

`NODEFAULT` marking a branch unreachable rather than trapping means a release build that *does* reach it has undefined behaviour rather than a crash. For a tool, that is the wrong trade; for a rebuild, trap.

## `to_native_text` (managed string to plain bytes)

**Contract** — converts a managed text object into a freshly allocated plain byte string and returns it. **The caller frees it.** The source buffer is pinned for the duration of the conversion so the collector cannot move it underneath the call.

```text
FUNCTION to_native_text(value : text) -> bytes      # caller owns the result
  pin value in place                                # the collector must not move it mid-call
  allocate (length + 1) * 2 bytes                   # worst case for the target encoding
  convert wide characters to the platform's narrow encoding
  FAIL WITH "conversion failed" IF the conversion reported an error
  RETURN the buffer
```

**Invariants** — the buffer is sized at twice the character count plus one, which is the worst case for the narrow encoding and is therefore always sufficient; it is not trimmed.

**Notes** — three real decisions hide in a nine-line function.

**Ownership crosses with the value.** The allocation is made with the plain system allocator, not the engine's pooled one, and the comment at the declaration says so — so it must be released with the matching system call, not the engine's. Mixing them is a silent corruption. This is the incidental shape of a decision a rebuild still has to make: **which allocator owns a string that was produced on one side of the boundary and is consumed on the other.**

**The conversion is lossy and nobody notices.** The narrow encoding is the platform's current one; a character outside it is replaced or dropped. Every weather identifier, every file name, every category label passes through here. In practice the editor's data is plain ASCII, which is why it has never mattered.

**A failed conversion is checked with the debug-only assertion above**, so in a release build a failure returns a partially converted buffer and execution continues. A rebuild should fail.

## `to_managed_text` (plain bytes to managed string)

**Contract** — wraps a plain byte string as a managed text object. Allocation is the runtime's, so nothing is owed back.

**Notes** — the asymmetry between the two directions — one obliges the caller, the other does not — is the whole ownership story of this boundary in miniature, and it is the single most likely place for a leak in a rebuild that keeps the split.
