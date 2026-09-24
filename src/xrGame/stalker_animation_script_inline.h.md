# src/xrGame/stalker_animation_script_inline.h

> Construction, copying and reading of a queued script animation — where the optional destination is actually made optional.

**Needs** — [`stalker_animation_script.h`](stalker_animation_script.h.md)
**Used by** — [`stalker_animation_script.h`](stalker_animation_script.h.md)
**Tier floor** — T3: field access and one conditional.

## Purpose

Holds the bodies for the record declared in
[`stalker_animation_script.h`](stalker_animation_script.h.md). Most of it is plain field
access; two things decide something and are written up here.

## Construction

**Contract** — takes the motion, the three flags and an optional destination transform.
When a destination is given it is copied into the record; when it is not, the record marks
itself as having none *and* fills the unused transform with a saturating sentinel value in
every component.

```text
FUNCTION make(animation, hand_usage, use_movement_controller, transform, local_animation)
  IF transform is given THEN
    self.transform := copy of transform
    self.has_transform := true
  ELSE
    self.has_transform := false
    self.transform := every component set to the largest representable magnitude
```

**Notes** — the sentinel fill is a debugging decision, not a functional one: if anything
reads the transform while `has_transform` is false, the creature is thrown so far from the
world that the bug is immediately visible rather than producing a plausible-looking pose at
the origin. A rebuild with a real `optional<Transform>` gets the same protection from the
type system and should not carry the sentinel across.

## Copying

**Contract** — copies every field, and then *re-points the record's destination reference
at its own copy of the transform* rather than at the source's.

**Invariants** — a copied record must not refer into the record it was copied from. The
source is typically a temporary being pushed onto the queue; keeping its address would
leave the queued entry pointing at freed storage the moment the push returned.

**Notes** — this is the whole reason the copy is written by hand. In a rebuild where the
destination is a value-typed `optional<Transform>`, the default copy is already correct and
this function disappears. It is listed here so that a reader of the original does not
mistake the hand-written copy for a semantic difference.

## `transform(object)`

**Contract** — returns the destination the animation should land at: the supplied transform
if there is one, otherwise the object's current world transform. Pure.

**Notes** — the fallback is what makes "play this animation where you stand" and "play this
animation ending exactly here" the same call site downstream. The caller never branches.

## `animation`, `hand_usage`, `use_movement_controller`, `local_animation`, `has_transform`

**Contract** — plain readers over the fields listed in
[`stalker_animation_script.h`](stalker_animation_script.h.md).
