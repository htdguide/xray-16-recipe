# src/xrCore/Debug/DXGetErrorString.inl

> The lookup table itself: roughly 3,200 numeric result codes mapped to their symbolic names, spanning the whole platform's error space and every graphics, audio and media subsystem the engine touches.

**Needs** — [`dxerr.cpp`](dxerr.cpp.md) · [`dxerr.h`](dxerr.h.md)
**Used by** — [`DXGetErrorDescription.inl`](DXGetErrorDescription.inl.md) · [`DXTrace.inl`](DXTrace.inl.md) · [`dxerr.cpp`](dxerr.cpp.md)
**Tier floor** — T1: the table's keys are a platform's packed 32-bit result values, matched by exact numeric identity.

## Purpose

This is the body of the name-lookup function, kept in its own file so it can be compiled twice with different string operations (see [`dxerr.cpp`](dxerr.cpp.md)). It is **data, not logic** — one enormous multi-way branch on the numeric code, returning a text literal — and its interest to a rebuild is entirely in its coverage and its failure behaviour, not its structure.

## `DXGetErrorString` — the body

**Contract** — given a result code, return the symbolic name it was declared under, as a pointer to a static literal. Never allocates, never fails, never blocks; returns the literal `Unknown` for a code not in the table. Thread-safe by virtue of being pure.

```text
FUNCTION name_of(code) -> text
  MATCH code
    ... roughly 3,200 exact-value arms ...
  RETURN "Unknown"
```

## Coverage

Grouped as the source groups it, because the groups are what a rebuild decides about:

| Group | Roughly |
|---|---|
| General platform error codes, component-object errors, and raw system error numbers | 2,900 entries — the overwhelming bulk |
| Legacy 2D graphics | 120 |
| The 2007-era 3D graphics interface, plus its later extensions | 40 |
| Audio | 30 |
| The 2008-era and 2009-era 3D graphics interfaces and their shared device layer | 40 |
| 2D vector graphics and text layout | 45 |
| Imaging codecs | 45 |
| The vendor sample framework | 10 |
| The newer audio engine and its effect layer | 5 |

**Invariants** — most system-error entries appear **twice**: once as the raw error number and once as that number packed into a result value. Both must map to the same name, because a code may arrive in either form depending on which layer returned it. See [`dxerr.cpp`](dxerr.cpp.md) for the packing rule.

A number of codes are aliases of each other and the source comments out the duplicates. That is not optional tidiness: two arms with the same value do not compile. A rebuild building this as a data table rather than a branch must deduplicate explicitly and pick which name wins — the source's choice is "the first one declared", and the choice is visible in every log line.

## What a rebuild should actually do

Almost none of this belongs in a rebuild as written. Two-thirds of the table is the general platform error space, which every modern platform can decode for itself; the value is in the third that it cannot — the graphics, audio and imaging codes. The honest shape is a small data file of the codes the engine's own seams can return, loaded or compiled in, and a fall-through to the platform's own decoder.

What must be preserved is the *contract*: a name is always returned, the caller never checks for failure, and the storage outlives the call. A rebuild returning an owned string changes every call site.

**Notes** — the names are the symbolic identifiers themselves, produced by stringifying them rather than by writing them out. That means the table cannot drift from the constants it names — a renamed constant renames its own log output — which is the one genuinely clever thing in the file and worth keeping in whatever form the rebuild's language offers.
