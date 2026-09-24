# src/xrGame/danger_object_inline.h

> The danger record's construction, its field reads, and the identity rule that makes a repeated perception one danger instead of many.

**Needs** — [`danger_object.h`](danger_object.h.md)
**Used by** — [`danger_object.h`](danger_object.h.md)
**Tier floor** — T3: field access and a comparison

## Purpose

The bodies of everything [`danger_object.h`](danger_object.h.md) declares. Separated for
inlining, which is a compilation concern; in a rebuild this is one type with one file.

## State

Adds nothing.

## Equality

**Contract** — the one piece of logic here, and the whole reason the file is worth reading.

```text
FUNCTION equals(a, b) -> bool
  IF a.object is absent XOR b.object is absent THEN RETURN false
  IF both present AND a.object.id != b.object.id THEN RETURN false
  # position, time and dependent object are deliberately not compared
  RETURN a.type == b.type AND a.perceive_type == b.perceive_type
```

**Notes** — comparing identifiers rather than object references is what lets a record
survive a save/load round trip and a re-spawn, and is the reason the manager can safely
keep records for entities it no longer holds a live reference to.
