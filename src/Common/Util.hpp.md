# src/Common/Util.hpp

> Two small conveniences — treating an enumeration as a bit set, and releasing a reference-counted device handle exactly once.

**Needs** — _(none)_
**Used by** — [`stdafx.h`](../xrCore/stdafx.h.md)
**Tier floor** — T1: the release helper exists because graphics resources are reference-counted across a foreign boundary, and the count must reach zero at a defined time.

## Purpose

Two unrelated habits that recur across the renderer and the engine, collected so they read
the same everywhere.

## State

Stateless.

## Enumeration-as-bit-set

**Contract** — declares that values of a named enumeration combine with bitwise or and
intersect with bitwise and, over a named integer width. The width is part of the
declaration because the value is stored and sometimes serialized at that width.

**Notes** — most languages either allow this on enumerations directly or provide a flag-set
type; the decision that survives is *which* enumerations are sets and what their underlying
width is, which each declaring file states.

## Release-once

**Contract** — given a handle to a reference-counted resource owned by the graphics device,
drop one reference and clear the handle. A cleared handle is ignored, so the same variable
may be released twice without harm; that tolerance is what makes it safe to call on every
field of a resource block during teardown, without first knowing which fields were ever
filled.

```text
FUNCTION release_once(handle)
  IF handle is none
    RETURN
  drop_one_reference(handle)
  handle <- none                # clearing is the whole point: teardown paths overlap
```

**Notes** — the underlying count belongs to the graphics driver, not to this engine — see
[Seam: Graphics device](../../SYSTEM-REQUIREMENTS.md#seam-graphics-device). A rebuild whose
graphics layer owns its resources with scopes deletes this helper; a rebuild that still
talks to a reference-counted driver keeps the idea and the tolerance of a null handle.

## Reference-count trace

**Contract** — in non-shipping builds only, report a resource's current reference count to the
log under a caller-supplied label, without changing it. It does so by taking a reference
and immediately dropping it, reading the count the drop returns. Compiled out of the
shipping build, where it must cost nothing.

**Notes** — this is a debugging instrument for finding resources that outlive the level that
created them. It is honest about being approximate: the count it reports includes
references the driver holds internally.
