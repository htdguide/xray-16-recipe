# src/xrGame/script_particle_action_inline.h

> The particle channel's constructors and setters — and the order dependence that decides whether the effect follows a bone or stands still.

**Needs** — [`script_particle_action.h`](script_particle_action.h.md)
**Used by** — nothing in this recipe; entry point or dead code.
**Tier floor** — T2

## Purpose

Bodies for [`script_particle_action.h`](script_particle_action.h.md). One real decision
lives here, and it is hidden in the order of two constructors' statements.

## The two constructors differ only in ordering, and that decides the goal kind

**Contract** —

```text
FUNCTION construct(effect_name, bone_name, placement, auto_remove)
  set_bone(bone_name)
  set_position(placement.position)     # tags the goal: positioned
  set_angles(placement.angles)
  set_velocity(placement.velocity)
  set_particle(effect_name, auto_remove)   # tags the goal: attached  <- wins

FUNCTION construct(effect_name, placement, auto_remove)
  set_particle(effect_name, auto_remove)   # tags the goal: attached
  set_position(placement.position)         # tags the goal: positioned <- wins
  set_angles(placement.angles)
  set_velocity(placement.velocity)
```

**Invariants** — the last tag-setting call wins, so the bone form ends *attached* and the
bone-less form ends *positioned*. That is the intended outcome and it is achieved entirely
by statement order, with nothing naming the intent. A rebuild should set the goal kind
explicitly from the constructor that knows which one it is, and treat the setters as
parameter writers only.

## Setters

**Contract** — `set_position` writes the position and tags the goal *positioned*;
`set_bone`, `set_angles` and `set_velocity` write their field only. All four clear both the
started flag and the completion flag, because any parameter change requires the effect to
be replayed rather than adjusted.

## `initialize`

**Contract** — does nothing. Present only because the action aggregate resets every channel
by the same name.
