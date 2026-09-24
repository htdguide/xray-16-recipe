# src/xrGame/ai/stalker/ai_stalker_inline.h

> The stalker's accessors: four sub-manager handles, a frame-stamp helper, and a hundred and eight one-line fire-queue readers.

**Needs** — [`ai_stalker.h`](ai_stalker.h.md)
**Used by** — [`ai_stalker.h`](ai_stalker.h.md)
**Tier floor** — T3: field reads

## Purpose

Accessors only. It is a separate file because the stalker's declaration is already
enormous and because these must be inline in the original for the fire-queue reads, which
happen per shot.

A rebuilder should read three things out of it and skip the rest.

**The four sub-manager accessors assert before returning.** Animation, planner, sight and
movement are each reached through an accessor that requires the handle to exist. They are
constructed in a fixed order and destroyed in another, and the assertions are what catches
an access from outside that window — which is a real hazard, because the planner's operators
call back into the stalker during construction.

**The frame-stamp helper** is the whole of the per-frame caching idiom used throughout the
stalker: pass a stored frame number, get back whether this is the first call this frame, and
have the number updated as a side effect. Every "compute at most once per frame" cache in
the stalker is this one function plus a field.

```text
FUNCTION frame_check(stored_frame) -> bool     # stored_frame is updated in place
  IF current_frame == stored_frame THEN RETURN false
  stored_frame = current_frame
  RETURN true
```

**A hundred and eight fire-queue readers**, one per (weapon class, range band, field), plus
ten range-boundary readers. They are mechanical field reads. A rebuild should replace the
whole block with an indexed lookup on (class, band) — the original's shape is a consequence
of C++ lacking a convenient way to name the tuple, not a decision.

## Exported units

- **the four sub-manager accessors** — animation, planner, sight, movement.
- **the per-frame stamp helper.**
- **flag and value accessors** — wounded, group behaviour, critical wound weights, throw
  state and interval, sniper update rate and fire mode, take-items and death-sound switches,
  the current best cover, the hit callback setter, the recoil effector, the display name.
- **the fire-queue and range-boundary readers.**
- **`throw_enabled`** — the one accessor with behaviour: it recomputes the grenade trajectory
  on demand when the cached one is stale, so a caller asking "can I throw" triggers the
  computation rather than reading a stale answer. That is the only lazily-computed accessor
  on the class and a rebuild must keep the laziness, because the trajectory check raycasts.
