# src/xrCore/Debug/DXGetErrorDescription.inl

> Asks the platform to explain a code first, and falls back to a table of about 300 hand-written sentences for the codes it cannot.

**Needs** — [`dxerr.cpp`](dxerr.cpp.md) · [`dxerr.h`](dxerr.h.md) · [`DXGetErrorString.inl`](DXGetErrorString.inl.md)
**Used by** — [`dxerr.cpp`](dxerr.cpp.md)
**Tier floor** — T1: a platform message service and a table keyed on packed result values.

## Purpose

The companion to [`DXGetErrorString.inl`](DXGetErrorString.inl.md): where that one gives a symbolic name, this gives a sentence. It is the one the crash reporter uses, because a name is for a developer and a sentence is for a bug report.

Its structure encodes one decision worth keeping. **The platform is asked first.** Only if the platform has nothing does the table apply — so the general error space, which the platform explains perfectly well and in the user's own language, is not duplicated here at all. That is why this table has about 300 entries where the name table has 3,200: it covers only the graphics, audio and imaging subsystems the platform does not know.

## `DXGetErrorDescription` — the body

**Contract** — write a description of the code into the caller's buffer, up to its capacity, always leaving it terminated. A capacity of zero writes nothing and returns. An unrecognized code leaves the buffer **empty**, not filled with a placeholder — which is a meaningful difference from the name lookup and the caller must handle it. Never allocates, never fails, never blocks.

```text
FUNCTION describe(code, buffer, capacity)
  IF capacity == 0 THEN RETURN
  buffer := empty
  n := ask the platform's message service to render this code into buffer,
       bounded by min(capacity, 32767), in the neutral locale
  IF n > 0 THEN RETURN                       # the platform knew it
  MATCH code
    ... roughly 300 arms, each copying a sentence into buffer ...
```

**Invariants** — the platform request is bounded at 32,767 characters regardless of how large the caller's buffer is, because the service's count parameter is narrower than the caller's. A rebuild whose message service takes a full-width count can drop the clamp; one that does not must keep it or truncate at a random point.

The neutral locale is requested explicitly. That is a decision, not a default: it means descriptions are **not** localized to the player's language, which makes them stable across machines and therefore comparable between bug reports. A rebuild that localizes them makes its own logs harder to triage.

**Notes** — the fall-through arms do not break, so control reaches the end of the switch after copying. That is harmless here and is an artifact of the two-flavour macro scheme in [`dxerr.cpp`](dxerr.cpp.md), which expands each arm to a bare copy statement.

The sentences themselves are the vendor's and carry no information a rebuild needs to preserve verbatim — unlike the *names* in the sibling file, which are matched against documentation. A rebuild may write its own, or may simply fall back to the symbolic name when the platform has nothing, which loses very little.
