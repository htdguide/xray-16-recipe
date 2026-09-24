# src/xrGame/smart_cover_animation_selector_inline.h

> The two accessors the smart-cover actions reach the planner and the world state through.

**Needs** — [`smart_cover_animation_selector.h`](smart_cover_animation_selector.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T3: accessors

## Purpose

Carries the bodies for
[`smart_cover_animation_selector.h`](smart_cover_animation_selector.h.md).

## `property_storage` / `planner`

**Contract** — field reads giving the outer planner's world state and the owned animation
planner. Both are how a running smart-cover action reaches back to the machinery that
scheduled it — to pin a property, or to read and write the dwell state. The planner
accessor asserts nothing; the planner is constructed with the selector and always exists.
