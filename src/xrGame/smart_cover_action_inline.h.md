# src/xrGame/smart_cover_action_inline.h

> The accessors on a loophole action, including the animation lookup that names the cover in its failure.

**Needs** — [`smart_cover_action.h`](smart_cover_action.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: field reads and one keyed lookup

## Purpose

Carries the bodies for [`smart_cover_action.h`](smart_cover_action.h.md).

## `movement` / `target_position`

**Contract** — field reads. `target_position` is only meaningful when `movement` is set;
nothing enforces that, and the only caller checks the flag first.

## `animations`

**Contract** — looks up the animation list for a named purpose and fails if there is none,
naming both the purpose and the cover it was looked for in. Returns the list itself, not a
copy; the caller picks from it.

**Invariants** — a missing purpose is an authoring error in shipped data, and the failure
message must carry the *cover's* identity as well as the purpose's, because the same
purpose name appears in dozens of covers and the message is the only way an author locates
the bad one. That is why the cover's name is threaded down as an argument rather than the
action holding a back-reference.
