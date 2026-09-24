# src/xrCore/Debug/dxerr.cpp

> Assembles four functions out of three shared bodies and two sets of string operations, so that one table of several thousand error codes serves both character widths.

**Needs** — [`dxerr.h`](dxerr.h.md) · [`DXGetErrorString.inl`](DXGetErrorString.inl.md) · [`DXGetErrorDescription.inl`](DXGetErrorDescription.inl.md) · [`DXTrace.inl`](DXTrace.inl.md)
**Used by** — [`DXGetErrorDescription.inl`](DXGetErrorDescription.inl.md) · [`DXGetErrorString.inl`](DXGetErrorString.inl.md) · [`DXTrace.inl`](DXTrace.inl.md) · [`dxerr.h`](dxerr.h.md)
**Tier floor** — T1: it decodes a platform's packed result values and composes them from vendor-specific facility and code fields.

## Purpose

The engine logs graphics and audio failures by code, and a code alone is useless in a bug report. This file is the decoder. It is an adopted vendor library, unchanged in substance, and almost all of its structure is an answer to a problem a rebuild does not have: *the same several-thousand-entry table must compile twice, once per character width, without being written twice.*

The answer is to put the three function bodies in separate files and include each of them twice, with the string operations bound differently each time. The three bodies get their own twins: [`DXGetErrorString.inl`](DXGetErrorString.inl.md), [`DXGetErrorDescription.inl`](DXGetErrorDescription.inl.md), [`DXTrace.inl`](DXTrace.inl.md). **This file itself decides nothing except the bindings**, and in a rebuild it vanishes entirely — one function per job, generic over its string type or simply committed to one.

## What does survive

Two things in this file are not incidental.

### Codes the platform does not define

Nine media-subsystem codes and ten framework codes are **composed here from their parts** rather than included from a header, because no header on the target defines them:

```text
code := (severity: error) | (facility) | (code)
```

with facilities drawn from the media subsystem's own numbering and a small block of framework codes numbered from 0x0901. They are constructed rather than declared because the framework they came from is a sample library that is not linked; the engine sees these codes come back from calls into components that *were* built against it. Reproducing them means reproducing the packing, which is the platform's, not this project's.

### The two ways a code can mean a system error

The table has entries that match a code **both as a raw system error number and as that number packed into a result value**. The packing is conditional: a value that is already negative is left alone, otherwise the low sixteen bits are taken, tagged with the system-error facility, and marked as a failure.

**Invariants** — this dual matching is why a single table entry catches a code whether it arrived from a call that returns raw error numbers or from one that returns packed results. A rebuild that keeps only one form will fail to decode roughly half the codes it sees in practice.

**Notes** — the three-thousand-byte buffer the trace body formats into is declared here as a shared constant rather than in the body that uses it, which is the kind of coupling the double-inclusion scheme forces. It is a diagnostic buffer; the size is generous and arbitrary.
