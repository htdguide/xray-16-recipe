# src/xrGame/smart_cover_action.h

> Declares one thing a creature can do while standing at a loophole: a named set of animation lists, and optionally a position the creature must move to first.

**Needs** — [`smart_cover_detail.h`](smart_cover_detail.h.md) · [`ai_monster_space.h`](ai_monster_space.h.md) · [Seam: Script virtual machine](../../SYSTEM-REQUIREMENTS.md#seam-script-virtual-machine)
**Used by** — [`smart_cover.cpp`](smart_cover.cpp.md) · [`smart_cover_action.cpp`](smart_cover_action.cpp.md) · [`smart_cover_action_inline.h`](smart_cover_action_inline.h.md) · [`smart_cover_loophole.cpp`](smart_cover_loophole.cpp.md) · [`smart_cover_loophole.h`](smart_cover_loophole.h.md) · [`smart_cover_loophole_planner_actions.cpp`](smart_cover_loophole_planner_actions.cpp.md)
**Tier floor** — T2: a parsed table of names

## Purpose

Declares the surface implemented in
[`smart_cover_action.cpp`](smart_cover_action.cpp.md) and
[`smart_cover_action_inline.h`](smart_cover_action_inline.h.md). An *action* is one of the
things a loophole offers — idle here, fire from here, reload here, look out from here. It
is nothing but data: which animation names serve which purpose, and whether taking the
action requires occupying a different spot.

## Exported units

- **The class** — a map from animation purpose to a list of interchangeable animation
  names, plus a movement flag and a target position.
- **`movement`** — whether this action places the creature somewhere other than the
  loophole's own spot. If it does, the cover precomputes a navigation vertex for that spot.
- **`target_position`** — that spot, in the cover object's local frame. Meaningful only
  when the movement flag is set.
- **`animations`** — the list of animation names for a named purpose; fails with both the
  purpose's and the cover's name when absent.
