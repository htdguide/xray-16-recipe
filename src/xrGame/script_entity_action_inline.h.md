# src/xrGame/script_entity_action_inline.h

> The bodies of the scripted-action type: channel assignment, per-channel completion, and the joint completion rule.

**Needs** — [`script_entity_action.h`](script_entity_action.h.md) · [`script_action_condition.h`](script_action_condition.h.md)
**Used by** — [`script_entity_action.h`](script_entity_action.h.md)
**Tier floor** — T2: predicate evaluation

## Purpose

Holds the implementations declared in
[`script_entity_action.h`](script_entity_action.h.md). The split is an artifact of needing
these inlined at every call site in the queue pump; the contracts and the joint completion
algorithm are documented in that header's twin and are not repeated here.

## Exported units

- `construct(other)` — copy an action.
- `set_action(channel)`, one per channel plus the opaque user value — replace a channel
  whole.
- `completed_movement`, `completed_watch`, `completed_animation`, `completed_sound`,
  `completed_particle`, `completed_object`, `completed_monster_action` — each reads the
  corresponding channel's own completion flag. They are separate names because script
  addresses them separately.
- `time_over` — has the action outlived its condition's lifetime.
- `completed` — the joint rule; see
  [`script_entity_action.h`](script_entity_action.h.md#completed--the-joint-completion-rule).
- `initialize` — reset the action and every channel.
- `move`, `look`, `anim`, `particle`, `object`, `cond`, `data` — channel readers.

**Notes**

Every per-channel completion test goes through one shared helper taking a channel by its
common base, which is the only thing the eight channel records have in common: a boolean
saying whether they are done. That base is the reason the aggregate works at all, and a
rebuild should keep it as an explicit interface rather than a base class.
