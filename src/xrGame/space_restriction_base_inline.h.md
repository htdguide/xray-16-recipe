# src/xrGame/space_restriction_base_inline.h

> The checked-build accessor for a restriction's border-connectivity verdict.

**Needs** — [`space_restriction_base.h`](space_restriction_base.h.md)
**Used by** — [`space_restriction_base.h`](space_restriction_base.h.md)
**Tier floor** — T4: one field read, present only in checked builds

## Purpose

One accessor, split out for C++ compilation reasons; fold into the type. It reads the flag
set by the connectivity self-test described in
[`space_restriction_base.cpp`](space_restriction_base.cpp.md).

## State

`Stateless.`

## `correct`

**Contract** — returns whether this restriction's border passed the flood-fill
connectivity test. Meaningful only after the border has been built. Exists only in checked
builds, and every reader of it is likewise checked-build-only.
