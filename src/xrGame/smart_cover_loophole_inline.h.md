# src/xrGame/smart_cover_loophole_inline.h

> The field reads on a loophole, plus the action-presence test the planner uses as a precondition.

**Needs** — [`smart_cover_loophole.h`](smart_cover_loophole.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: accessors and one keyed lookup

## Purpose

Carries the bodies for [`smart_cover_loophole.h`](smart_cover_loophole.h.md).

## Geometry and limits

**Contract** — `id`, `fov`, `danger_fov`, `range`, `fov_position`, `fov_direction`,
`danger_fov_direction`, `enter_direction` and `actions` are plain reads. The vectors are in
the cover object's local frame and the angles are in radians, already converted from the
degrees the author wrote.

## `usable`

**Contract** — a read of a flag derived at parse time, not a computation. It is false
exactly when the loophole has no actions.

## `enterable` / `exitable`

**Contract** — read and write. Both are written once by the description's post-load pass
([`smart_cover_description.cpp`](smart_cover_description.cpp.md)) from the presence of an
edge to or from the reserved outside vertices, and are read-only thereafter.

## `is_action_available`

**Contract** — whether a named action exists at this loophole. This is the test the
smart-cover planner evaluates as a precondition before offering an action, so it must be a
lookup and not a scan of the action's contents.
